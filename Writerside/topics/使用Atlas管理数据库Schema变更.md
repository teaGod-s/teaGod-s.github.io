# 使用 Atlas 管理数据库 Schema 变更

## 数据库 Schema 变更管理痛点 {id="atlas_1"}

后端开发时经常会涉及到表结构变更，表结构变更是重要且高危的操作，所以必须要有审核过程。

一般后端的上线流程是这样，会先通过 SQL 审计平台提交 SQL 工单，经过层层审批将表结构变更同步到生产环境后，再上线业务代码。
比如 [Yearning](https://github.com/cookieY/Yearning)、[Archery](https://github.com/hhyo/Archery) 这些平台，就支持 SQL 工单审核。

但是当你的业务库越来越庞大，每次上线都要提交几十上百个工单的时候，不管是对开发者还是对审批者来说，都是灾难。

更不用说当你忘记提交某个工单，却直接把代码上线了。那等来的将是上线事故，滋味可是相当不好受(￣_￣|||)


## 数据库 Migration 工具 Atlas 介绍 {id="atlas_2"}

数据库 Migration（迁移）是什么？ 数据库 Migration（迁移） 是指对数据库 Schema（结构）进行版本化管理和增量变更的过程。简单来说，就是像管理代码一样管理数据库结构的变化。

针对上面介绍的数据库 Schema 变更管理痛点，我们是不是可以将期望的表结构与业务代码放在一起，用 Git 一起管理，然后用一种数据库 Migration 工具，在每次代码上线时，将 Git 里保存的表结构，自动同步到生产环境上呢？

可以的，今天介绍的 [Altas](https://github.com/ariga/atlas)，就是这样的工具。

Atlas 同时支持[**声明式工作流**](https://atlasgo.io/declarative/apply)与[**版本化工作流**](https://atlasgo.io/versioned/intro)两种流程，是一个非常强大的工具。我们接下来依次介绍。

## 安装 Atlas {id="atlas_3"}
我推荐使用 [mise](https://mise.en.dev/)来安装并管理 Atlas ，因为它支持 Atlas 的多版本管理。而且通过 mise task 对 Atlas 命令进行自定义封装后，可以极大简化使用复杂度。

有关 mise 的安装使用可以参考 <a href="Windows系统的Golang多版本管理.md" as="button">我的这篇博客</a>

安装过程很简单，在项目根目录下的 mise.toml 中定义好如下代码，然后执行 `mise install`
```toml
[tools]
atlas = "1.3.0"
```


## Atlas 声明式工作流 {id="atlas_4"}

声明式工作流讲究的是所见即所得。 在代码库中定义好建表语句后，后面每次字段变更时不需要再单独写 `ALTER` 语句了，直接改建表语句就行。

Atlas 会将你声明好的最新版建表语句直接同步到目标数据库实例上。

这种声明式工作流思想在 K8s 中也有用到。比如用 YAML 文件描述好应用的期望状态，K8s 会负责将应用按照你声明好的 YAML 文件部署到集群上，而你不需要关注具体过程。

那么接下来介绍一下声明式工作流怎么使用。

首先来看一下我的代码结构，我的项目采用的是 go-zero 框架。

```shell
├── app
│   ├── gateway
│   │   ...
│   ├── order
│   │   ├── client
│   │   │   ...
│   │   ├── etc
│   │   │   ├── order.yaml
│   │   │   └── order_pro.yaml
│   │   ├── internal
│   │   │   ├── config
│   │   │   │   ...
│   │   │   ├── handler
│   │   │   │   ...
│   │   │   ├── logic
│   │   │   │   ...
│   │   │   ├── model
│   │   │   │   ├── order_items_model.go
│   │   │   │   ├── order_items_model_gen.go
│   │   │   │   ├── orders_model.go
│   │   │   │   ├── orders_model_gen.go
│   │   │   │   └── vars.go
│   │   │   ├── server
│   │   │   │   ...
│   │   │   ├── svc
│   │   │   │   └── service_context.go
│   │   │   └── types
│   │   │       └── types.go
│   │   ├── order.api
│   │   ├── order.go
│   │   ├── order.proto
│   │   ├── pb
│   │   │   ...
│   │   └── sql
│   │       ├── migrations
│   │       └── schema
│   │           ├── order_item.sql
│   │           └── orders.sql
│   └── user
│   │   ...
├── atlas.hcl
├── doc
│   ...
├── go.mod
├── go.sum
├── kit
│   ...
└── mise.toml
```

可以看到 order 服务的所有建表语句都放在 `app/order/sql/schema` 目录下，目前只有两张表。

那么如何使用 Atlas 将 `orders.sql` 和 `order_item.sql` 同步到目标库中呢？很简单，一条命令就行

```shell
atlas schema apply \                          
  --url "mysql://root:123456@localhost:3307/wsl_test" \
  --to file://app/order/sql/schema \
  --dev-url "docker://mysql/8/dev"
```

其中每个参数的作用如下：
- 参数 url 为目标数据库地址
- 参数 to 为存放建表语句即 schema 声明的本地位置
- 参数 dev-url 为一个与业务无关的、临时的、隔离的数据库实例，Atlas 将其用作沙箱来模拟真实环境。

可以看到 dev-url 参数是 Atlas 实现数据库自动化 Migration 的关键。

在本条命令执行后， Atlas 会干4件事

1. Atlas 会调用 docker 拉起一个临时数据库实例，然后将本地定义好的 schema 即建表语句先应用到 dev-url 所在的临时库中。
2. 随后 Atlas 会拿 dev-url 所在实例的 schema 与 url 所在目标库实例的 schema 做对比计算。计算出来的差异部分 Atlas 会生成 SQL 工单来供我们审批， 你可以选择通过或者是拒绝。
3. 如果审批通过，Atlas 会自动将差异 SQL 放在目标库实例上执行，最终达成的效果是目标库的 schema 与本地代码仓库声明的 schema 保持一致。 
4. 命令执行完后重置 dev-url 所在实例的 schema。由于在本例中使用的是 docker，所以临时拉起的 docker 容器会自动销毁。

接下来我们来做个实验：

第一步：首先在目标库中新建4张表
<include from="公用sql代码片段.topic" element-id="e_commerce_system_table"></include>

第二步：修改本地代码库中的 `orders.sql` 和 `order_item.sql` 两个文件
<tabs>
    <tab title="orders.sql">
        <code-block lang="sql">
            CREATE TABLE `orders` (
              `id` bigint unsigned NOT NULL AUTO_INCREMENT COMMENT '订单ID',
              `user_id` bigint unsigned NOT NULL DEFAULT 0 COMMENT '用户ID',
              `order_amount` decimal(10,2) NOT NULL DEFAULT 0 COMMENT '订单金额',
              `paid_amount` decimal(10,2) NOT NULL DEFAULT 0 COMMENT '支付金额',
              `pay_time` datetime DEFAULT NULL COMMENT '支付时间',
              PRIMARY KEY (`id`),
              KEY `idx_user_id` (`user_id`)
            ) ENGINE=InnoDB COMMENT='订单表';
        </code-block>
    </tab>
    <tab title="order_item.sql">
        <code-block lang="sql">
            CREATE TABLE `order_items` (
               `id` bigint unsigned NOT NULL AUTO_INCREMENT COMMENT '详情ID',
               `order_id` bigint unsigned NOT NULL DEFAULT 0 COMMENT '订单ID',
               `product_id` bigint unsigned NOT NULL DEFAULT 0 COMMENT '商品ID',
               `quantity` int unsigned NOT NULL DEFAULT 0 COMMENT '商品数量',
               PRIMARY KEY (`id`),
               KEY `idx_order_id` (`order_id`),
               KEY `idx_product_id` (`product_id`)
            ) ENGINE=InnoDB COMMENT='订单详情表';
        </code-block>
    </tab>
</tabs>

可以看到 代码库中的两个 sql 文件与目标库的表结构基本一样，唯独多了 `DEFAULT 0`，但仅仅如此也会造成表结构差异。
幸运的是 Atlas 可以检查到这点，并生成对应的 `ALTER` 语句。

第三步：在本地代码库根目录新建一个 `atlas.hcl` 文件，填入如下代码
```hcl
# 定义开发测试用的临时沙箱数据库（Dev Database），与业务无关
# PS:因为我本地测试时使用 docker 一直失败，所以这里改成了我自己创建的一个空白库
variable "dev_url" {
  type    = string
  default = "mysql://root:123456@127.0.0.1:3307/altas_dev"
}
# 1. 本地开发环境（声明式 Schema 模式）
env "order_local" {
  url = "mysql://root:123456@127.0.0.1:3307/wsl_test" # 目标数据库
  src = "file://app/order/sql/schema" # Schema 声明文件路径
  dev = var.dev_url # 计算 diff 需要 Dev 数据库
}
```
我们在这个 `atlas.hcl` 文件里，把刚才的 Atlas 命令参数给持久化了，并且跟项目代码一起用 Git 管理。

它就像 docker compose 中的 `docker-compose.yml` 文件一样，会简化我们的基础命令，

第四步：接下来我们执行
```shell
atlas schema apply --env order_local
```
结果如下所示，Atlas 会等待我们审批，是拒绝（Abort）还是通过（Approve）

![atlas 声明式 migration 默认版](atlas_declarative_with_drop.png)

但是我们发现 Atlas 除了生成 `orders` 和 `order_items` 两张表的 `ALTER` 语句之外，把我们另外两张表（`products` 和 `users`）也 `DROP` 掉了。

`DROP` 语句可是一个非常高危的操作！！！生产环境这么用可是要完蛋。所以我们接下来要拦截一下 `DROP` 语句。

第五步： 拒掉（Abort）刚才的工单。

第六步：再次修改 `atlas.hcl` 文件，将 `order_local` 代码块改成下面这样：
```HCL
# 1. 本地开发环境（声明式 Schema 模式）
env "order_local" {
  url = "mysql://root:123456@127.0.0.1:3307/wsl_test" # 目标数据库
  src = "file://app/order/sql/schema" # Schema 声明文件路径
  dev = var.dev_url # 计算 diff 需要 Dev 数据库
  diff {
      skip {
          drop_table  = true # 忽略 drop 等高危操作
          drop_schema = true
      }
  }
}
```
第七步：在控制台执行如下命令
```shell
atlas schema apply --env order_local --dry-run
```
加上 dry-run 参数后代表该命令为“试运行”，所以不会生成实际的工单，也不会对目标库执行任何写入操作。

![atlas 声明式 migration 过滤 drop 版](atlas_declarative_without_drop.png)

可以看到现在只剩下 `ALTER` 语句了。接下来把 dry-run 参数去掉，再次执行命令，并选择审批通过（Approve），结果如下所示
```shell
➜  hello-zero git:(master) ✗ atlas schema apply --env order_local          
Planning migration statements (2 in total):

  -- modify "order_items" table:
    -> ALTER TABLE `order_items` MODIFY COLUMN `order_id` bigint unsigned NOT NULL DEFAULT 0 COMMENT "订单ID", MODIFY COLUMN `product_id` bigint unsigned NOT NULL DEFAULT 0 COMMENT "商品ID", MODIFY COLUMN `quantity` int unsigned NOT NULL DEFAULT 0 COMMENT "商品数量";
  -- modify "orders" table:
    -> ALTER TABLE `orders` MODIFY COLUMN `user_id` bigint unsigned NOT NULL DEFAULT 0 COMMENT "用户ID", MODIFY COLUMN `order_amount` decimal(10,2) NOT NULL DEFAULT 0.00 COMMENT "订单金额", MODIFY COLUMN `paid_amount` decimal(10,2) NOT NULL DEFAULT 0.00 COMMENT "支付金额";

-------------------------------------------                                                                                                        
                                                                                                                                                   
Analyzing planned statements (2 in total):                                                                                                         

  -- no diagnostics found

  -------------------------
  -- 119.850006ms
  -- 2 schema changes

-------------------------------------------                                                                                                        
                                                                                                                                                   
Applying approved migration (2 statements in total):

  -- modify "order_items" table
    -> ALTER TABLE `order_items` MODIFY COLUMN `order_id` bigint unsigned NOT NULL DEFAULT 0 COMMENT "订单ID", MODIFY COLUMN `product_id` bigint unsigned NOT NULL DEFAULT 0 COMMENT "商品ID", MODIFY COLUMN `quantity` int unsigned NOT NULL DEFAULT 0 COMMENT "商品数量";
  -- ok (47.962508ms)

  -- modify "orders" table
    -> ALTER TABLE `orders` MODIFY COLUMN `user_id` bigint unsigned NOT NULL DEFAULT 0 COMMENT "用户ID", MODIFY COLUMN `order_amount` decimal(10,2) NOT NULL DEFAULT 0.00 COMMENT "订单金额", MODIFY COLUMN `paid_amount` decimal(10,2) NOT NULL DEFAULT 0.00 COMMENT "支付金额";
  -- ok (22.286367ms)

  -------------------------
  -- 70.423409ms
  -- 1 migration
  -- 2 sql statements
```
整个声明式工作流就跑通了，以后改动表结构完全不用手写 `ALTER` 语句，怎么样是不是很方便？

另外由于我使用的是 go-zero 框架，model 层代码也是由 sql 文件生成的。

所以我每次想改表结构只需改 sql 文件就行，model 层代码与目标库 schema 全都靠自动化工具自己搞定，省时又省力，开发体验直接起飞~

<warning>建议实际应用中一定要加上第六步的约束，这项才能避免 `DROP` 误操作带来的生产事故！！！</warning>

## Atlas 版本化工作流 {id="atlas_5"}