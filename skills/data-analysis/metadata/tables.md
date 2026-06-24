# 表元数据

MySQL 表与 Tushare API 源、SQL schema 文件、字段详情的映射关系。

---

## stock_info_ts

| 属性 | 值 |
|------|-----|
| **描述** | Tushare 股票信息表，含申万行业分类 |
| **SQL 文件** | `data/stock_info/stock_info.sql` |
| **Tushare API** | `stock_basic` + `index_classify` + `index_member_all` |
| **MCP 工具** | `mcp__tushare__stock_basic` |
| **唯一键** | `(day, ts_code)` |
| **主要列** | `ts_code`, `name`, `area`, `industry`, `market`, `exchange`, `list_date`, `delist_date`, `status`, `l1_code`/`l1_name`, `l2_code`/`l2_name`, `l3_code`/`l3_name` |

### Tushare 字段映射

| MySQL 列 | Tushare API 字段 | 说明 |
|----------|-----------------|------|
| `ts_code` | `ts_code` | 来自 `stock_basic` |
| `name` | `name` | 股票名称（含 ST 标识） |
| `area` | `area` | 所属地域 |
| `industry` | `industry` | Tushare 原始行业分类，非申万行业 |
| `fullname` | `fullname` | 股票全称 |
| `enname` | `enname` | 英文全称 |
| `cnspell` | `cnspell` | 拼音缩写 |
| `market` | `market` | 市场类型（主板/创业板/科创板/北交所） |
| `exchange` | `exchange` | SSE 上交所 / SZSE 深交所 / BSE 北交所 |
| `curr_type` | `curr_type` | 交易货币 |
| `status` | `list_status` → 映射: L→1, D→0, P→2, G→3 | 上市状态 |
| `list_date` | `list_date` | 上市日期 |
| `delist_date` | `delist_date` | 退市日期 |
| `is_hs` | `is_hs` | 沪深港通标的：N 否 / H 沪股通 / S 深股通 |
| `act_name` | `act_name` | 实控人名称 |
| `act_ent_type` | `act_ent_type` | 实控人企业性质 |
| `l1_code` ∼ `l3_name` | 来自 `index_classify` (SW2021) + `index_member_all` | `is_new='Y'` 为最新行业分类 |
| `day` | — | 系统字段：取数日期 |
| `qlib_code` | — | 由 `ts_code` 衍生 |

### 注意事项
- `status` 取值：1=上市, 0=退市, 2=暂停上市, 3=过会未交易
- `is_new='Y'` 标识每只股票最新的申万行业分类

---

## trade_daily_ts

| 属性 | 值 |
|------|-----|
| **描述** | 日线行情数据（原始数据），含 OHLCV 及复权因子 |
| **SQL 文件** | `data/trade_daily/trade_daily.sql` |
| **Tushare API** | `daily` + `adj_factor` |
| **MCP 工具** | `mcp__tushare__daily`, `mcp__tushare__adj_factor` |
| **唯一键** | `(day, ts_code)` |
| **主要列** | `ts_code`, `day`, `open`, `close`, `high`, `low`, `pre_close`, `change`, `pct_chg`, `vol`, `amount`, `adj_factor` |

### Tushare 字段映射

| MySQL 列 | Tushare API 字段 | 说明 |
|----------|-----------------|------|
| `ts_code` | `ts_code` | 来自 `daily` |
| `day` | `trade_date` | 交易日期 |
| `open` | `open` | 开盘价 |
| `close` | `close` | 收盘价 |
| `high` | `high` | 最高价 |
| `low` | `low` | 最低价 |
| `pre_close` | `pre_close` | 昨日收盘价 |
| `change` | `change` | 收盘价涨跌额 |
| `pct_chg` | `pct_chg` | 收盘价涨跌幅，单位：% |
| `vol` | `vol` | 成交量，单位：手（×100 股） |
| `amount` | `amount` | 成交额，单位：千元 |
| `adj_factor` | `adj_factor` | 来自 `adj_factor` API，在 `daily` 获取后合并 |
| `qlib_code` | — | 由 `ts_code` 衍生 |

### 注意事项
- `amount` != `close * vol`——amount 是不同成交价下的实际成交金额
- `adj_factor` 通过 `pro.adj_factor()` 单独获取，按 `(ts_code, trade_date)` 合并
- 单位：vol 为手（×100 股），amount 为千元（÷1000 得元）

---

## valuation_ts

| 属性 | 值 |
|------|-----|
| **描述** | Tushare 市值数据——每日估值指标（PE、PB、市值等） |
| **SQL 文件** | `data/valuation_ts/valuation_ts.sql` |
| **Tushare API** | `daily_basic` |
| **MCP 工具** | `mcp__tushare__daily_basic` |
| **唯一键** | `(day, ts_code)` |
| **主要列** | `ts_code`, `day`, `close`, `turnover_rate`, `turnover_rate_f`, `volume_ratio`, `pe`, `pe_ttm`, `pb`, `ps`, `ps_ttm`, `dv_ratio`, `dv_ttm`, `total_share`, `float_share`, `free_share`, `total_mv`, `circ_mv` |

### Tushare 字段映射

| MySQL 列 | Tushare API 字段 | 说明 |
|----------|-----------------|------|
| `ts_code` | `ts_code` | TS 股票代码 |
| `day` | `trade_date` | 交易日期 |
| `close` | `close` | 当日收盘价 |
| `turnover_rate` | `turnover_rate` | 换手率，单位：% |
| `turnover_rate_f` | `turnover_rate_f` | 换手率（自由流通股），单位：% |
| `volume_ratio` | `volume_ratio` | 量比 |
| `pe` | `pe` | 市盈率，亏损公司为 NULL |
| `pe_ttm` | `pe_ttm` | 市盈率（TTM），亏损公司为 NULL |
| `pb` | `pb` | 市净率 |
| `ps` | `ps` | 市销率 |
| `ps_ttm` | `ps_ttm` | 市销率（TTM） |
| `dv_ratio` | `dv_ratio` | 股息率，单位：% |
| `dv_ttm` | `dv_ttm` | 股息率（TTM），单位：% |
| `total_share` | `total_share` | 总股本，单位：万股 |
| `float_share` | `float_share` | 流通股本，单位：万股 |
| `free_share` | `free_share` | 自由流通股本，单位：万股 |
| `total_mv` | `total_mv` | 总市值，单位：万元 |
| `circ_mv` | `circ_mv` | 流通市值，单位：万元 |
| `qlib_code` | — | 由 `ts_code` 衍生 |

### 注意事项
- `pe` 和 `pe_ttm` 对亏损公司为 NULL——不是数据质量问题
- 单位转换：股本为万股、市值为万元——实际值需 ×10000
- 该表也包含 `close` 字段（与 `trade_daily_ts.close` 同源）

---

## shibor

| 属性 | 值 |
|------|-----|
| **描述** | 上海银行间同业拆放利率（Shanghai Interbank Offered Rate） |
| **SQL 文件** | `data/rate/rate.sql` |
| **Tushare API** | `shibor` |
| **MCP 工具** | `mcp__tushare__shibor` |
| **唯一键** | `(date)` |
| **主要列** | `date`, `on_rate`, `1w`, `2w`, `1m`, `3m`, `6m`, `9m`, `1y` |

### Tushare 字段映射

| MySQL 列 | Tushare API 字段 | 说明 |
|----------|-----------------|------|
| `date` | `date` | 日期 |
| `on_rate` | `on` | 隔夜利率，单位：% |
| `1w` | `1w` | 1 周利率，单位：% |
| `2w` | `2w` | 2 周利率，单位：% |
| `1m` | `1m` | 1 个月利率，单位：% |
| `3m` | `3m` | 3 个月利率，单位：% |
| `6m` | `6m` | 6 个月利率，单位：% |
| `9m` | `9m` | 9 个月利率，单位：% |
| `1y` | `1y` | 1 年利率，单位：% |

### 注意事项
- 所有利率均已年化处理（%），应为正值
- 无 `ts_code`——这是市场级表，非个股级别
- MySQL 查询中使用反引号包裹列名：`` `1w` ``、`` `1m` `` 等

---

## income_ts

| 属性 | 值 |
|------|-----|
| **描述** | Tushare 利润表 |
| **SQL 文件** | `data/income/income.sql` |
| **Tushare API** | `income` |
| **MCP 工具** | `mcp__tushare__income` |
| **唯一键** | `(end_date, ts_code, f_ann_date, update_flag)` |
| **主要列** | `ts_code`, `ann_date`, `f_ann_date`, `end_date`, `report_type`, `comp_type`, 65 个财务科目, `update_flag` |

### Tushare 字段映射

65 个财务科目字段与 Tushare `income` API 一一对应（字段名完全一致）。关键非财务字段：

| MySQL 列 | Tushare API 字段 | 说明 |
|----------|-----------------|------|
| `ts_code` | `ts_code` | TS 股票代码 |
| `ann_date` | `ann_date` | 公告日期（首次发布报告日期） |
| `f_ann_date` | `f_ann_date` | 实际公告日期（修改后报告发布日期） |
| `end_date` | `end_date` | 报告期（会计周期终止日期） |
| `report_type` | `report_type` | 1=一季报, 2=半年报, 3=三季报, 4=年报 |
| `comp_type` | `comp_type` | 1=一般工商业, 2=银行, 3=保险, 4=证券 |
| `update_flag` | `update_flag` | 0=已被取代, 1=最新 |

### 注意事项
- **已知问题：** 存在 `update_flag=0` 但 `ann_date != f_ann_date` 的数据（Tushare 记录差异）
- 财务科目取决于 `comp_type`——银行/保险/证券的列填列情况不同
---

## balance_ts

| 属性 | 值 |
|------|-----|
| **描述** | Tushare 资产负债表 |
| **SQL 文件** | `data/balance/balance.sql` |
| **Tushare API** | `balancesheet` |
| **MCP 工具** | `mcp__tushare__balancesheet` |
| **唯一键** | `(end_date, ts_code, f_ann_date, update_flag)` |
| **主要列** | `ts_code`, `ann_date`, `f_ann_date`, `end_date`, `report_type`, `comp_type`, 120+ 个财务科目, `update_flag` |

### Tushare 字段映射

所有财务科目字段与 Tushare `balancesheet` API 一一对应（字段名完全一致）。

### 注意事项
- **已知重复行：** `920491.BJ` 在 `(end_date=2018-12-31, update_flag=1)` 和 `(end_date=2021-06-30, update_flag=1)` 各存在 2 行完全重复——属于 Tushare 源数据问题，非 MySQL ETL 缺陷
- 财务科目取决于 `comp_type`——一般工商业为标准科目，保险/证券有额外特定科目

---

## cashflow_ts

| 属性 | 值 |
|------|-----|
| **描述** | Tushare 现金流量表 |
| **SQL 文件** | `data/cashflow/cashflow.sql` |
| **Tushare API** | `cashflow` |
| **MCP 工具** | `mcp__tushare__cashflow` |
| **唯一键** | `(end_date, ts_code, f_ann_date, update_flag)` |
| **主要列** | `ts_code`, `ann_date`, `f_ann_date`, `end_date`, `comp_type`, `report_type`, 80+ 个现金流科目, `update_flag` |

### Tushare 字段映射

所有现金流科目字段与 Tushare `cashflow` API 一一对应（字段名完全一致）。

### 注意事项
