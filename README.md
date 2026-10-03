# shop-faker

[![Python](https://img.shields.io/badge/python-3.12%2B-blue.svg)](https://www.python.org/)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Managed by uv](https://img.shields.io/badge/managed%20by-uv-261230.svg)](https://docs.astral.sh/uv/)

电商仿真数据生成器。按可控的规模与分布，生成一套跨表一致、时间线自洽的电商交易数据与用户行为埋点，用于搭建 CDP 平台、离线数仓、实时链路等大数据项目的练习数据集。

## 目录

- [定位](#定位)
- [数据域](#数据域)
- [真实性设计](#真实性设计)
- [输出布局](#输出布局)
- [安装](#安装)
- [使用](#使用)
- [配置](#配置)
- [下游对接](#下游对接)
- [标准文档](#标准文档)
- [项目结构](#项目结构)
- [开发](#开发)
- [参与贡献](#参与贡献)
- [许可](#许可)

## 定位

字段填满只是第一步。生成的数据要真正能用于建模，还需要在业务语义上站得住：订单能关联到真实存在的商品，埋点能回溯到对应的成交，用户画像前后一致，金额可以逐单复算。这是本项目的重点。

具体来说，生成的每一份数据满足：

- 任意订单明细都能关联到真实存在的 SKU 与商品，金额可复算
- 每个用户的行为序列会真实地导向成交
- 订单状态流转严格遵循状态机，时间戳单调递增且间隔合理
- 用户消费分层、商品销量、活跃时段等符合真实分布，而非均匀随机

不适用的场景：需要真实个人信息的场景（全部为合成数据）、需要生成恶意/污染数据的测试（本项目不产生脏数据）。

## 数据域

### 交易域

关系型结构，共 12 个实体，表间通过外键关联，可导出为 JSON Lines 与 SQL 两种形态。字段口径以《[交易域数据标准说明书 V1.0](docs/交易域数据标准说明书V1.0.md)》为准。

| 表 | 说明 | 主键 | 关键外键 |
| --- | --- | --- | --- |
| `user` | 用户主档，含注册渠道、地域、会员等级 | `user_id` | — |
| `user_address` | 收货地址，一个用户多条，含默认地址标记 | `address_id` | `user_id` |
| `shop` | 店铺，含主营类目、开店时间、评分 | `shop_id` | `main_category_id` |
| `category` | 类目树，自关联多级 | `category_id` | `parent_id` |
| `product` | 商品 SPU，含标价、品牌、上架时间 | `product_id` | `shop_id`, `category_id` |
| `sku` | 商品 SKU，含规格、售价、成本 | `sku_id` | `product_id` |
| `cart` | 购物车，一行一个用户的一个 SKU | `cart_id` | `user_id`, `shop_id`, `product_id`, `sku_id` |
| `order` | 订单主表，含金额拆分与状态 | `order_id` | `user_id`, `shop_id`, `address_id` |
| `order_item` | 订单明细，一行一个 SKU | `item_id` | `order_id`, `product_id`, `sku_id` |
| `payment` | 支付流水，含渠道与渠道流水号 | `payment_id` | `order_id` |
| `refund` | 退款单，含原因与退款金额 | `refund_id` | `order_id`, `payment_id` |
| `logistics` | 物流单，含承运商、运单号与签收时间 | `delivery_id` | `order_id` |

金额口径统一为：`order.pay_amount = order.order_amount − order.discount_amount + order.freight_amount`，且 `order.order_amount = Σ order_item.item_amount`，生成后由校验器逐单复算，不允许出现对不上的订单。完整不变量见标准说明书第 7 节。

### 行为埋点域

事件型结构，每条一个 JSON 对象，按小时分区。事件口径与字段定义以《[行为埋点标准口径说明书 V1.0](docs/行为埋点标准口径说明书V1.0.md)》为准，下表为落库时的扁平化表示。

| 字段 | 说明 |
| --- | --- |
| `eventId` | 事件唯一 ID |
| `eventType` | 事件类型，取值见口径说明书第 5 节 |
| `eventTime` | 事件实际发生时间，ISO 8601 带时区 |
| `userId` | 登录用户 ID，未登录行为为 `null` |
| `deviceId` | 设备 ID，用于串联匿名与登录态 |
| `sessionId` | 会话 ID，30 分钟无事件自动切分 |
| `pageId` | 页面标识 |
| `productId` / `skuId` | 关联商品，非商品类事件为 `null` |
| `properties` | 事件私有属性，JSON 对象 |

`baseInfo` 中的公共字段（`appId`、`appVersion`、`platform` 及设备、地理位置等）随每条事件一并落库，扁平化后与事件属性处于同一层。

事件按用户漏斗组织，同一会话内顺序不倒退：

```
app_launch → page_view → product_expose → product_click → product_view → cart_add → order_submit → order_pay
```

旁支事件：`search_submit`、`favorite_add`、`favorite_remove`、`cart_remove`、`product_share`、`button_click`、`page_leave`、`app_exit`、`order_cancel`、`order_refund`。

事件不在触发时立即落盘，按口径说明书的缓存策略组装成报文：缓存事件数达到 `cacheMaxLength`，或首条事件的等待时长达到 `cacheWaitingTime` 时，将缓存中的事件组成一个报文写出。单个报文的事件数不超过 `cacheMaxLength`。

漏斗各环节的跃迁概率可配置，用于压出不同的转化率。

## 真实性设计

从四个维度保证数据的业务可信度。

### 跨表业务一致性

- 用户画像在全部表中保持稳定，年龄、地域、注册时间不会自相矛盾
- 商品的店铺归属、类目归属、价格区间在 SKU 与订单明细间保持传递一致
- 订单地址必须是下单用户本人持有的地址；下单时间必须在注册时间之后
- 金额逐单复算，`order_item` 汇总与 `order` 汇总必须闭合

### 统计分布真实

默认使用真实分布而非均匀分布，权重均可在配置中替换：

| 维度 | 分布 |
| --- | --- |
| 用户年龄 | 分段正态，主体落在 22–40 岁 |
| 用户地域 | 按省级行政区常住人口加权 |
| 用户消费能力 | 帕累托分布，头部约 5% 用户贡献 40% 以上 GMV |
| 商品销量 | Zipf 分布，长尾明显 |
| 活跃时段 | 双峰，午间 11–13 点与夜间 20–23 点为峰值 |
| 客单价 | 按类目分别取对数正态 |
| 复购间隔 | 对数正态，按用户活跃度分层 |

### 时间线与状态机

订单状态流转受状态机约束，非法跃迁不会出现。状态取值与流转规则见数据标准说明书第 6、7 节：

```
10 待付款 ──支付成功──> 20 已付款 ──发货──> 30 已发货 ──签收──> 40 已签收
    │                      │
    └──取消/超时──> 50 已取消  └──申请退款──> 60 退款中 ──退款完成──> 70 已退款
```

每一跳的时延从配置的分布中采样，而非固定值：提交订单到支付成功通常数分钟，支付到发货数小时至一天，发货到签收一到数天，跨境或偏远地区自动拉长。

行为事件与订单事件共享同一条时间线：一次下单必然能在埋点中回溯到对应的 `order_submit` 与 `order_pay` 事件，且埋点的 `eventTime` 与交易域的 `order.order_time` 偏差在合理区间内。

### 行为关联与生命周期

用户不是每天都均匀下单。生成器为每个用户维护一个生命周期状态，状态之间按马尔可夫链迁移：

```
新客 → 活跃 → 沉默 → 流失
        ↑______|
       (召回)
```

状态决定该用户当日的活跃概率、下单概率与会话数。由此自然产生复购间隔变化、部分用户长期沉默、少量用户高频活跃等真实特征。时间跨度足够长时，可以观察到完整的用户生命周期曲线。

## 输出布局

```
output/
├── ods/                              # 交易域，按天分区
│   ├── user/dt=2026-01-01/part-00000.jsonl
│   ├── order/dt=2026-01-01/part-00000.jsonl
│   ├── order_item/dt=2026-01-01/part-00000.jsonl
│   └── ...
├── event/                            # 埋点域，按小时分区
│   └── behavior_event/dt=2026-01-01/hour=12/part-00000.jsonl
├── sql/
│   ├── ddl.sql                       # 建表语句
│   └── data.sql                      # INSERT 语句，可直接灌入关系库
└── manifest.json                     # 本次生成的元信息与行数统计
```

维度表（`user`、`shop`、`product`、`sku`、`category`、`user_address`）按首次出现日期分区，`--snapshot-mode` 可切换为每日全量快照，用于练习拉链表的构建。

## 安装

需要 Python 3.12 及以上，以及 [uv](https://docs.astral.sh/uv/)。

```bash
git clone https://github.com/jiuyue1123/shop-faker.git
cd shop-faker
uv sync
```

作为全局命令安装：

```bash
uv tool install .
shop-faker --help
```

## 使用

### 离线批量生成

```bash
uv run shop-faker batch \
  --config config/demo.toml \
  --start-date 2026-01-01 \
  --end-date 2026-03-31 \
  --output output/
```

命令行参数覆盖配置文件中的同名项。未显式指定的参数一律使用配置文件的值，项目不提供隐式默认规模——规模必须显式给出，避免误跑出几个 TB。

先干跑一次确认规模：

```bash
uv run shop-faker plan --config config/demo.toml
```

`plan` 会打印各表预计行数、预计落盘体积与分区数量，不实际生成数据。

### 实时流式生成

```bash
uv run shop-faker stream \
  --config config/demo.toml \
  --rate 2000 \
  --duration 1h
```

`stream` 复用同一套生成内核，按指定的每秒事件数持续产出，写入按小时切分的目录，模拟实时链路的落盘行为。`--accelerate N` 可按 N 倍速回放历史数据，用于让 Flink 作业在几分钟内消费完一天的事件量。

批量与流式共用同一份配置和同一套模型定义，两者生成的实体结构与字段语义完全一致，切换模式不需要改动下游。

### 导出 SQL

```bash
uv run shop-faker dump-sql \
  --config config/demo.toml \
  --dialect mysql \
  --output output/sql/
```

生成 `ddl.sql` 与 `data.sql`，`data.sql` 为分批 INSERT，可直接：

```bash
mysql -u root -p shop < output/sql/ddl.sql
mysql -u root -p shop < output/sql/data.sql
```

支持的方言：`mysql`、`postgresql`。批量 INSERT 的批大小由 `--batch-size` 控制。

### 校验生成结果

```bash
uv run shop-faker verify --input output/
```

对已生成的数据重新执行全部一致性断言（金额闭合、引用完整性、状态机合法、时间单调、状态与时间字段一致），断言清单即数据标准说明书第 7 节的业务不变量，用于确认产出可用。

## 配置

配置为 TOML 格式，见 `config/demo.toml`。规模相关项没有内置默认值，必须显式配置。

```toml
[global]
seed        = 20261003        # 随机种子，相同种子产出相同数据
start_date  = "2026-01-01"
end_date    = "2026-03-31"
locale      = "zh_CN"
timezone    = "Asia/Shanghai"

[scale]
user     = 200_000
shop     = 5_000
product  = 80_000
order    = 1_000_000          # 目标订单量，实际值受转化率约束后收敛

[behavior]
dau_ratio                = 0.35   # 日活用户占存量比
sessions_per_user        = 2.5
events_per_session       = 18
conversion.cart_add      = 0.12   # product_view → cart_add
conversion.order_submit  = 0.35   # cart_add → order_submit
conversion.order_pay     = 0.78   # order_submit → order_pay

[upload]
cacheMaxLength   = 5    # 最大缓存事件数，达到即立即触发发送
cacheWaitingTime = 5    # 延迟发送时长，秒；超时未满则发送已缓存事件

[order]
pay_success_rate   = 0.78
cancel_rate        = 0.09
refund_rate        = 0.06
aov_mu             = 4.6      # 客单价对数正态的 mu
aov_sigma          = 0.85

[lifecycle]
new_user_ratio     = 0.08     # 每日新增用户占比
churn_prob         = 0.004    # 活跃 → 流失 的日迁移概率
revive_prob        = 0.02     # 沉默 → 活跃 的日迁移概率

[output]
format      = "jsonl"         # jsonl | json
compression = "none"          # none | gzip | zstd
partition   = "day"           # day | hour
```

分布参数（`aov_mu`、`aov_sigma` 等）直接对应统计分布的参数，改配置即可压出不同的数据画像，不需要改代码。

## 下游对接

**离线数仓**：`output/ods` 与 `output/event` 均采用 Hive 风格分区路径，可直接：

```sql
CREATE EXTERNAL TABLE ods_order (...)
PARTITIONED BY (dt STRING)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
LOCATION 'hdfs:///data/shop-faker/ods/order';

MSCK REPAIR TABLE ods_order;
```

Spark 读取：

```python
spark.read.schema(schema).json("output/ods/order")          # 自动识别 dt= 分区
```

**实时链路**：`stream` 模式配合 `--accelerate`，可让 Flink/Spark Streaming 作业在压缩后的时间内消费完整历史事件，用于调试窗口、水位线与状态后端。

**CDP 建模**：交易域提供用户的事实行为（订单、支付、退款），埋点域提供行为轨迹，两者通过 `userId` 与时间线对齐，足以支撑 RFM 分层、用户标签体系、人群圈选、漏斗分析与留存计算等典型 CDP 场景。会员与营销域暂未覆盖。

## 标准文档

`docs/` 下两份说明书定义了数据的字段口径与业务约束，是生成器的实现依据：

| 文档 | 内容 |
| --- | --- |
| [交易域数据标准说明书 V1.0](docs/交易域数据标准说明书V1.0.md) | 12 个实体的字段构成、类型规范、枚举字典、业务不变量与校验规则 |
| [行为埋点标准口径说明书 V1.0](docs/行为埋点标准口径说明书V1.0.md) | 18 个事件的触发时机、属性口径、上报结构、缓存批量策略与验收用例 |

两份文档冲突时以交易域数据标准为准。

## 项目结构

```
shop-faker/
├── src/shop_faker/
│   ├── cli.py                # Typer 命令入口
│   ├── config.py             # 配置模型与加载
│   ├── models/               # Pydantic 数据模型
│   │   ├── dimension.py      # user / user_address / shop / category / product / sku
│   │   ├── transaction.py    # cart / order / order_item / payment / refund / logistics
│   │   └── event.py          # behavior_event
│   ├── generators/           # 生成器内核
│   │   ├── population.py     # 用户与商品存量初始化，含分布采样
│   │   ├── lifecycle.py      # 用户状态机与活跃度演化
│   │   ├── order.py          # 订单与状态机推进
│   │   └── behavior.py       # 埋点漏斗与会话切分
│   ├── sinks/                # 输出适配
│   │   ├── jsonl.py
│   │   └── sql.py
│   ├── distributions.py      # 分布采样工具
│   └── verify.py             # 一致性校验
├── config/
│   └── demo.toml
├── docs/
│   ├── 交易域数据标准说明书V1.0.md
│   └── 行为埋点标准口径说明书V1.0.md
└── tests/
```

## 开发

```bash
uv run pytest                        # 单元测试
uv run pytest -m slow                # 包含规模压测
uv run ruff check .                  # 静态检查
uv run ruff format .
uv run mypy src/
```

新增业务表时，先在 `models/` 定义 Pydantic 模型并声明外键约束，再在 `generators/` 实现生成逻辑，最后在 `verify.py` 补充对应的断言，并同步更新 `docs/` 下对应的标准说明书。模型定义是外键校验与 SQL DDL 生成的唯一来源。

## 参与贡献

欢迎提交 Issue 与 Pull Request。改动生成逻辑前请先确认：

- 新增或修改数据模型时，同步更新 `verify.py` 中的一致性断言，并补充对应测试
- 涉及分布参数的改动，在 PR 描述中附上改动前后的数据画像对比
- 提交前本地跑通 `uv run pytest` 与 `uv run ruff check .`

## 许可

MIT
