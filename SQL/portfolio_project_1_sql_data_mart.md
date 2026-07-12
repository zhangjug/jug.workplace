# 作品 1：Fab Production Logistics KPI Data Mart

## 1. 项目定位

**英文项目名：** Fab Production Logistics KPI Data Mart  
**中文项目名：** 晶圆厂生产物流 KPI 数据集市与产能监控看板

这个作品的目标不是证明你了解半导体工艺细节，而是证明你能用物流工程、SQL 和数据建模能力，把制造系统中的 lot 流转数据转化为 Industrial Engineering 可用的产能指标、瓶颈诊断和管理看板。

适配岗位关键词：

| JD 能力要求 | 作品对应能力 |
|---|---|
| factory capacity modeling | 建立 tool / tool group 产能日历与负载模型 |
| performance metrics tracking | 计算 WIP、throughput、cycle time、queue time、utilization |
| SQL / Excel / data visualization | 用 SQL 建表、建视图、做 dashboard |
| audit manufacturing capacity information | 建立 capacity audit exception 规则 |
| data and report consultant | 把指标组织成业务可读的看板和分析结论 |
| communicate data findings | 在 README 中输出 IE 视角的业务发现 |

## 2. 数据地址

主数据源：

- GitHub 仓库：[Job Shop Scheduling Benchmark: Environments and Instances](https://github.com/ai-for-decision-making-tue/Job_Shop_Scheduling_Benchmark_Environments_and_Instances)
- 数据目录：[data/jsp](https://github.com/ai-for-decision-making-tue/Job_Shop_Scheduling_Benchmark_Environments_and_Instances/tree/main/data/jsp)
- 推荐起步数据：[data/jsp/adams](https://github.com/ai-for-decision-making-tue/Job_Shop_Scheduling_Benchmark_Environments_and_Instances/tree/main/data/jsp/adams)
- 对应论文：[Job Shop Scheduling Benchmark: Environments and Instances](https://arxiv.org/abs/2308.12794)

建议先使用以下实例：

- `abz5`
- `abz6`
- `abz7`
- `abz8`
- `abz9`

这些实例规模适中，适合做第一个 SQL 作品。原始数据是 job-shop scheduling benchmark，不是真实半导体 fab 数据，因此 README 中需要明确写：

> This project builds a fab-like production logistics data mart from public job-shop scheduling benchmark instances.

## 3. 数据映射逻辑

公开数据中的 job-shop scheduling 概念可以映射为 fab 生产物流场景：

| Benchmark 概念 | Fab-like 场景概念 | 说明 |
|---|---|---|
| job | lot / wafer lot | 一批待加工产品 |
| operation | process step | lot 的一道工序 |
| machine | tool / tool group | 可加工该工序的设备 |
| processing time | run time | 设备加工时间 |
| operation sequence | route sequence | 工艺路线顺序 |
| waiting before operation | queue time | 进入工站后的等待时间 |
| schedule start / end | production event | 生产事件开始与结束 |
| completed job | completed lot | 完成全部工序的 lot |

这个映射对物流工程背景是合理的，因为核心问题是多工序、多资源、排队、产能、瓶颈和周期时间，而不是半导体物理工艺本身。

## 4. 推荐项目目录

```text
fab-production-logistics-sql/
  README.md
  data/
    raw/
      abz5
      abz6
      abz7
      abz8
      abz9
    processed/
      dim_lot.csv
      dim_tool.csv
      dim_operation.csv
      fact_schedule_event.csv
      fact_capacity_calendar.csv
  scripts/
    01_parse_benchmark_to_csv.py
  sql/
    01_schema.sql
    02_load_data.sql
    03_kpi_views.sql
    04_analysis_queries.sql
  dashboard/
    screenshots/
  docs/
    data_dictionary.md
    methodology.md
```

## 5. 最终交付物

建议最终至少完成这些内容：

1. 一个 GitHub 项目仓库。
2. 一份清晰的 README。
3. 一组处理后的 CSV 数据。
4. 一套 SQL 建表脚本。
5. 一套 SQL KPI view。
6. 一套业务分析 SQL 查询。
7. 一页 dashboard 截图。
8. 一段可以直接放进简历的项目描述。

## 6. 创建步骤

### 第 1 步：确定项目包装

预计用时：0.5 天

项目定位：

> Build a SQL-based production logistics data mart to monitor WIP, throughput, cycle time, queue time, tool utilization, bottlenecks, and capacity audit exceptions in a fab-like manufacturing system.

README 第一段要说明三件事：

1. 这是一个 fab-like manufacturing system，不是真实企业 fab 数据。
2. 数据来自公开 job-shop scheduling benchmark。
3. 项目重点是 IE 产能监控和生产物流 KPI，不是算法竞赛。

### 第 2 步：下载并理解原始数据

预计用时：0.5 天

下载 GitHub 仓库后，先看：

```text
data/jsp/adams/abz5
data/jsp/adams/abz6
data/jsp/adams/abz7
data/jsp/adams/abz8
data/jsp/adams/abz9
```

JSP 数据通常表示为：

- 第一行：job 数量、machine 数量。
- 后续每一行：一个 job 的工艺路线。
- 每个 job 的路线由多组 `machine_id processing_time` 组成。

目标是把原始数据解析为 operation-level 表：

| instance_id | lot_id | operation_seq | tool_id | run_time |
|---|---:|---:|---:|---:|
| abz5 | L001 | 1 | T04 | 88 |
| abz5 | L001 | 2 | T09 | 68 |

### 第 3 步：生成 fab-like event log

预计用时：1 天

原始 benchmark 主要提供路线和加工时间，不一定直接提供每道工序的开始时间和结束时间。为了做 KPI，需要生成一张模拟生产事件表。

推荐使用简单 FIFO dispatching 规则：

1. 每个 lot 按工艺路线顺序加工。
2. 每台 tool 同一时间只能加工一个 operation。
3. operation 的开始时间取决于：
   - 当前 lot 上一道工序完成时间；
   - 当前 tool 可用时间。
4. 若 tool 不可用，则 lot 进入等待状态，产生 queue time。

需要生成的字段：

| 字段 | 含义 |
|---|---|
| event_id | 事件 ID |
| instance_id | 数据实例，例如 `abz5` |
| lot_id | lot 编号 |
| operation_id | 工序 ID |
| operation_seq | lot 的第几道工序 |
| tool_id | 设备编号 |
| tool_group | 设备组 |
| queue_start_ts | 进入等待时间 |
| start_ts | 开始加工时间 |
| end_ts | 完成时间 |
| queue_hours | 等待时间 |
| run_hours | 加工时间 |
| due_ts | 计划交期 |
| priority | lot 优先级 |
| status | completed / running / waiting |

建议生成这些 CSV：

```text
dim_lot.csv
dim_tool.csv
dim_operation.csv
fact_schedule_event.csv
fact_capacity_calendar.csv
```

### 第 4 步：设计数据库表

预计用时：0.5-1 天

推荐数据库：

- 首选：PostgreSQL
- 快速版本：SQLite

核心表：

```sql
dim_lot
dim_tool
dim_operation
fact_schedule_event
fact_capacity_calendar
```

表设计建议：

#### dim_lot

| 字段 | 类型 | 说明 |
|---|---|---|
| lot_id | text | lot 编号 |
| instance_id | text | 数据实例 |
| release_ts | timestamp | lot 释放时间 |
| due_ts | timestamp | 交期 |
| priority | integer | 优先级 |
| product_family | text | 产品族，可模拟为 NAND_A / NAND_B |

#### dim_tool

| 字段 | 类型 | 说明 |
|---|---|---|
| tool_id | text | 设备编号 |
| tool_group | text | 设备组 |
| tool_type | text | 设备类型 |
| is_constraint_tool | boolean | 是否瓶颈候选设备 |

#### dim_operation

| 字段 | 类型 | 说明 |
|---|---|---|
| operation_id | text | 工序编号 |
| lot_id | text | lot 编号 |
| operation_seq | integer | 工序顺序 |
| tool_id | text | 目标设备 |
| planned_run_hours | numeric | 计划加工时间 |

#### fact_schedule_event

| 字段 | 类型 | 说明 |
|---|---|---|
| event_id | text | 事件 ID |
| lot_id | text | lot 编号 |
| operation_id | text | 工序编号 |
| tool_id | text | 设备编号 |
| queue_start_ts | timestamp | 开始等待时间 |
| start_ts | timestamp | 开始加工时间 |
| end_ts | timestamp | 完成时间 |
| queue_hours | numeric | 等待小时 |
| run_hours | numeric | 加工小时 |
| status | text | 状态 |

#### fact_capacity_calendar

| 字段 | 类型 | 说明 |
|---|---|---|
| calendar_date | date | 日期 |
| shift | text | 班次 |
| tool_id | text | 设备编号 |
| available_hours | numeric | 可用产能小时 |
| planned_downtime_hours | numeric | 计划停机小时 |

`fact_capacity_calendar` 是作品亮点，因为它把调度数据和 capacity modeling 连接起来。

### 第 5 步：编写 SQL KPI views

预计用时：1-1.5 天

至少完成 6 个 view。

#### 1. v_lot_cycle_time

业务问题：

> 每个 lot 从 release 到 finish 花了多久？等待时间占比是多少？

输出字段：

- lot_id
- release_ts
- finish_ts
- total_cycle_hours
- total_queue_hours
- total_run_hours
- queue_ratio
- due_ts
- on_time_flag

#### 2. v_daily_throughput

业务问题：

> 每天完成多少 lot 和 operation？throughput 是否稳定？

输出字段：

- production_date
- completed_lots
- completed_operations
- avg_cycle_hours
- avg_queue_hours
- avg_run_hours

#### 3. v_tool_utilization_daily

业务问题：

> 哪些设备利用率过高，可能成为约束资源？

输出字段：

- production_date
- tool_id
- tool_group
- run_hours
- available_hours
- utilization_rate

#### 4. v_wip_by_tool_group

业务问题：

> WIP 堆积在哪些 tool group？

输出字段：

- snapshot_ts
- tool_group
- waiting_wip
- running_wip
- total_wip

#### 5. v_bottleneck_ranking

业务问题：

> 当前最可能的瓶颈工站是哪一个？

建议综合三个指标：

- utilization rate
- average queue time
- waiting WIP

可定义：

```text
bottleneck_score = utilization_score * 0.4
                 + queue_time_score * 0.4
                 + wip_score * 0.2
```

输出字段：

- tool_group
- utilization_rate
- avg_queue_hours
- waiting_wip
- bottleneck_score
- bottleneck_rank

#### 6. v_capacity_audit_exceptions

业务问题：

> 哪些产能或流转异常需要 IE 关注？

异常规则：

- utilization_rate > 90%
- queue_hours > P90 queue time
- capacity_gap > 0
- total_cycle_hours > target_cycle_hours
- due date missed
- route step missing

输出字段：

- exception_id
- exception_type
- lot_id
- tool_id
- tool_group
- severity
- metric_value
- threshold_value
- detected_at

### 第 6 步：编写业务分析 SQL

预计用时：0.5 天

建议写 8-10 条分析查询，放入：

```text
sql/04_analysis_queries.sql
```

查询题目建议：

1. Which tool group is the current bottleneck?
2. Which lots have the highest delay risk?
3. What percentage of cycle time is waiting time?
4. Which tools are over 90% utilized?
5. Which process steps create the longest queues?
6. What is the daily throughput trend?
7. Where is capacity shortage most severe?
8. Which lots missed due date and why?
9. Which tool group has the highest WIP accumulation?
10. How much additional capacity is required to reduce bottleneck load below 85%?

SQL 里尽量使用：

- `GROUP BY`
- `JOIN`
- `CASE WHEN`
- `CTE`
- window function
- percentile / ranking logic

这些比简单查询更能体现 SQL 能力。

### 第 7 步：制作 dashboard

预计用时：1 天

工具任选一个：

- Power BI：最适合简历展示。
- Tableau Public：适合公开展示。
- Metabase：偏数据产品。
- Streamlit：适合做轻量交互网页。

推荐 dashboard 一页完成，结构如下：

#### 顶部 KPI cards

- Total WIP
- Daily Throughput
- Avg Cycle Time
- Avg Queue Time
- Bottleneck Tool Group
- On-time Rate

#### 中间图表

- WIP by tool group
- Daily throughput trend
- Tool utilization heatmap
- Queue time by operation
- Bottleneck ranking

#### 底部明细表

- Capacity audit exceptions
- Delay-risk lots

Dashboard 截图建议放入：

```text
dashboard/screenshots/main_dashboard.png
```

### 第 8 步：撰写 README

预计用时：0.5-1 天

README 建议结构：

```text
1. Business Background
2. Data Source
3. Fab-like Data Mapping
4. Database Schema
5. KPI Definitions
6. SQL Highlights
7. Dashboard Screenshots
8. Key Findings
9. Limitations and Next Steps
```

Key Findings 可以写成模拟业务结论：

- Tool group `TG_03` had the highest bottleneck score due to high utilization and long queue time.
- Waiting time contributed more than 60% of average cycle time.
- Capacity shortage was concentrated in late-stage operations.
- A small number of high-utilization tools caused most delay-risk lots.

Limitations 要诚实写：

- 数据来自公开 benchmark，不是真实 fab MES 数据。
- due date、priority、capacity calendar 为模拟生成。
- 项目重点是 IE 指标系统设计，不是精确复刻企业生产系统。

## 7. 预计总用时

| 阶段 | 预计用时 |
|---|---:|
| 选题包装 + 下载数据 | 0.5 天 |
| 解析数据 + 生成 event log | 1 天 |
| 建数据库表 | 0.5-1 天 |
| 写 KPI views | 1-1.5 天 |
| 写分析 SQL | 0.5 天 |
| 做 dashboard | 1 天 |
| 写 README + 简历 bullet | 0.5-1 天 |

总计：

- 快速可展示版本：2-3 天
- 简历可用版本：5-7 天
- 面试可深讲版本：7-10 天

## 8. 简历写法

英文版：

> Built a SQL-based fab production logistics data mart using public job-shop scheduling benchmark data; designed KPI views for WIP, throughput, cycle time, queue time, tool utilization, bottleneck ranking, and capacity audit exceptions.

中文版：

> 基于公开 Job Shop Scheduling benchmark 构建类晶圆厂生产物流数据集市，使用 SQL 设计 WIP、产出、周期时间、排队时间、设备利用率、瓶颈识别和产能异常审计指标，并制作可视化看板支持 IE 产能分析。

## 9. 面试讲法

建议用 60 秒说明：

> 我的专业是物流工程，所以我从生产物流和产能流动角度切入这个岗位。我用公开 job-shop scheduling benchmark 模拟 fab lot 在不同 tool group 之间的流转，先把原始路线和加工时间转成 event log，再用 SQL 建立数据集市，计算 WIP、throughput、cycle time、queue time、tool utilization 和 bottleneck score。最后做了一页 dashboard，用于识别产能异常和延迟风险。这个项目主要体现我把制造数据转化为 IE 决策指标的能力。

