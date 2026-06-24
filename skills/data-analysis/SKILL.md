---
name: data-analysis
description: 分析 MySQL 数据质量——缺失值、异常值统计，以及 MySQL 与 Tushare API 数据交叉校验。当用户要求检查数据质量、分析数据完整性、发现异常值或验证 MySQL 数据与 Tushare 源数据一致性时使用。
---

# 数据分析 Skill

对 cf_quant 数据库中的核心表进行数据质量分析。使用 **MySQL MCP**（`mcp__mysql__execute_sql`）查询本地数据，使用 **Tushare MCP**（`mcp__tushare__*`）获取源数据进行交叉校验。

## 何时使用

- "分析 trade_daily_ts 的缺失值"
- "对比 MySQL 和 Tushare 数据是否一致"
- "检查 valuation_ts 的异常值"
- "生成数据质量报告"
- 任何关于这些表数据完整性、准确性或异常的问题

## 准备工作

分析前始终加载表元数据：本 skill 同目录下的 metadata/tables.md。

该文件包含：表描述、SQL schema 文件路径、Tushare API 名称、唯一键、每张表的已知问题。

## 分析流程

### 1. 缺失值分析

对指定表及可选的日期/股票过滤条件：

**第一步：逐列缺失率。** 对每个数值/值列统计 NULL 数量和缺失比例。跳过系统列（`id`）。对于财务报表表（`income_ts`、`balance_ts`、`cashflow_ts`），必须指定 `end_date` 范围——全表扫描数据量过大。

```sql
SELECT
  COUNT(*) AS total,
  SUM(CASE WHEN col IS NULL THEN 1 ELSE 0 END) AS nulls,
  ROUND(SUM(CASE WHEN col IS NULL THEN 1 ELSE 0 END) / COUNT(*) * 100, 2) AS null_pct
FROM table_name
WHERE ... -- 日期/股票过滤条件
```

**第二步：缺失模式分析。** 检查：
- **按股票完整性：** 哪些股票缺失行数最多？（日线表：对比每只股票的行数与预期交易日数）
- **按日期完整性：** 哪些日期的行数少于预期？（对比行数与 `stock_info_ts` 中当日存活股票数）
- **连续缺失检测：** 对于日线表，检查缺失值是否聚集（例如同一股票连续多日缺失大概率是停牌/退市，而非数据质量问题）
- **财务报表特殊处理：** 检查 `update_flag` 分布——0 表示被同一 `(end_date, ts_code, f_ann_date)` 下的 1 所取代。不适用科目缺失财务字段 属于正常现象。

**第三步：原因推断。** 解读分析结果：
- 某股票日线数据缺失 → 检查 `stock_info_ts.status`（是否停牌/退市？）、`list_date`（是否尚未上市？）
- 财务字段缺失 → 检查 `comp_type`（银行/保险/证券的利润表科目不同）
- PE/PB 为空 → 亏损公司（正常现象）
- stock_info_ts 中缺少 `l1_code/l2_code/l3_code` → 早期上市股票，在申万行业分类建立之前

### 2. 异常值分析

**第一步：逐列统计。** 对数值列计算：
```sql
SELECT
  MIN(col), MAX(col), AVG(col), STDDEV(col)
FROM table_name WHERE ...
```

**第二步：异常值检测方法**（根据上下文选择）：
- **价格数据**（`open/close/high/low`）：检查零值/负值，涨跌幅是否超过涨跌停限制（主板 10%、创业板/科创板 20%、北交所 30%）
- **估值比率**（`pe/pb/ps`）：大于 1000 或小于 -1000 的值可疑；PE 为负 = 亏损公司（不一定是错误）
- **成交量/成交额**：成交量为 0 但价格有变动 → 可疑
- **Z-score/IQR(分位数Q3-Q1)**：超出 5σ 或 3 倍 IQR 的值标记为极端异常
- **跨列一致性检查**：`high >= max(open, close)`、`low <= min(open, close)`、`close * vol != amount`（正常——amount 是实际成交均价 × 手数）

**第三步：汇总。** 按严重程度分组报告异常值：
- **严重**（确认数据错误）：负价格、high < low 等
- **警告**（可疑）：极端 PE、成交量 0 但有涨跌
- **提示**（可能正常）：亏损公司的负 PE

### 3. MySQL 与 Tushare 交叉校验

实时对比 MySQL 数据与 Tushare API 源数据。

**第一步：确定校验范围。** 询问用户：
- 待校验的表
- 日期范围或具体日期
- 抽样数量（默认：单日校验全部行，多日随机抽取 20 只股票）

**第二步：双源查询。**
- MySQL：`SELECT * FROM table_name WHERE ...`
- Tushare：调用对应的 MCP 工具（映射关系见 tables.md）。参数对应：`ts_code` → `ts_code`，`trade_date`/`end_date` → 日期列。

**第三步：逐字段比对。**
- 按唯一键对齐行（见 tables.md）
- 对每个共有字段，按容差比较：
  - `decimal` 列：`abs(mysql - tushare) < 1e-5` 视为相等（允许舍入误差）
  - `date` 列：必须完全匹配
  - `varchar` 列：必须完全匹配（名称大小写不敏感）
- 跳过仅限 MySQL 的衍生字段：`id`（自增主键）、`qlib_code`（由 `ts_code` 衍生）、`day`/`date`（用于关联，无需比对）

**第四步：差异报告。**
- 明确不一致：列出行键 + 字段名 + 两边值
- MySQL 缺失但 Tushare 存在：数据尚未同步
- Tushare 没有但 MySQL 存在：过期数据（已退市股票、历史记录）

### 4. 财务报表特殊处理

`income_ts`、`balance_ts`、`cashflow_ts` 共享相同的去重键：`(end_date, ts_code, f_ann_date, update_flag)`。
- **已知问题（balance_ts）：** `920491.BJ` 存在 2 行完全重复的数据（2018-12-31、2021-06-30）——两个完全相同的 `(end_date, ts_code, f_ann_date, update_flag=1)` 记录
- **校验提示：** 对比财务报表数据时，先过滤 `update_flag=1`

## 报告模板

分析结束时生成结构化 Markdown 报告。如用户要求则保存为文件：

```markdown
# 数据质量报告：{表名}
**日期：** {YYYY-MM-DD}
**范围：** {日期范围 / 过滤条件}

## 1. 概览
- 总行数：{N}
- 日期范围：{开始} 至 {结束}
- 股票数量：{M}

## 2. 缺失值
| 字段 | 总数 | NULL数 | NULL% | 评估 |
|------|------|--------|-------|------|
| ...  | ...  | ...    | ...   | 正常/警告/异常 |

### 缺失模式
- {模式描述}

## 3. 异常值
| 字段 | 值 | 股票 | 日期 | 严重程度 | 说明 |
|------|-----|------|------|----------|------|
| ...  | ... | ...  | ...  | 严重/警告/提示 | ... |

## 4. 交叉校验（如已执行）
| 键 | 字段 | MySQL | Tushare | 状态 |
|----|------|-------|---------|------|
| ... | ... | ... | ... | 一致/不一致 |

- 一致率：{X}%
- 发现差异：{N} 处

## 5. 总结
- 整体数据质量：良好/一般/较差
- 待处理事项：{列表}
```

## 注意事项

- 全表扫描前先预估行数：`SELECT COUNT(*) FROM table` 并加合适的过滤条件
- 大表（trade_daily_ts：1400 万行，valuation_ts：1400 万行）必须先缩小范围——询问用户日期范围
- `information_schema.TABLES.TABLE_ROWS` 是近似值（InnoDB）；用 `SELECT COUNT(*)` 获取精确行数
- `qlib_code` 列是衍生字段，不来自 Tushare——交叉校验时跳过
- `stock_info_ts.day` 是取数日期而非交易日——用 `trade_cal` 校验交易日历
