# 部署后设置

本文描述了您在部署 StarRocks 之后需要执行的任务。

在将新的 StarRocks 集群投入生产之前，您必须管理初始帐户并设置必要的变量和属性以使集群正常运行。

## 管理初始帐户

创建 StarRocks 集群后，系统会自动生成集群的初始 `root` 用户。`root` 用户拥有 `root` 权限，即集群内所有权限的集合。我们建议您修改 `root` 用户密码并避免在生产中使用该用户，以避免误用。

1. 使用用户名 `root` 和空密码通过 MySQL 客户端连接到 StarRocks。

   ```Bash
   # 将 <fe_address> 替换为您连接的 FE 节点的 IP 地址（priority_networks）
   # 或 FQDN，将 <query_port> 替换为您在 fe.conf 中指定的 query_port（默认：9030）。
   mysql -h <fe_address> -P<query_port> -uroot
   ```

2. 执行以下 SQL 重置 `root` 用户密码：

   ```SQL
   -- 将 <password> 替换为您要为 root 用户设置的密码。
   SET PASSWORD = PASSWORD('<password>')
   ```

> **说明**
>
> - 重置密码后请务必妥善保管。如果您忘记了密码，请参阅 [重置丢失的 root 密码](../administration/User_privilege.md#重置丢失的-root-密码) 了解详细说明。
> - 完成部署后设置后，您可以创建新用户和角色来管理团队内的权限。有关详细说明，请参阅 [管理用户权限](../administration/User_privilege.md)。

## 设置必要的系统变量

为使您的 StarRocks 集群在生产环境中正常工作，您需要设置以下系统变量：

| **变量名**                          | **StarRocks 版本** | **推荐值**                                                   | **说明**                                                     |
| ----------------------------------- | ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| is_report_success                   | v2.4 或更早        | false                                                        | 是否发送查询 Profile 以供分析。默认值为 `false`，即不发送。将此变量设置为 `true` 会影响 StarRocks 的并发性能。 |
| enable_profile                      | v2.5 或以后        | false                                                        | 是否发送查询 Profile 以供分析。默认值为 `false`，即不发送。将此变量设置为 `true` 会影响 StarRocks 的并发性能。 |
| enable_pipeline_engine              | v2.3 或以后        | true                                                         | 是否启用 Pipeline Engine。`true` 表示启用，`false` 表示禁用。默认值为 `true`. |
| parallel_fragment_exec_instance_num | v2.3 或以后        | 如果您启用了 Pipeline Engine，您可以将此变量设置为`1`。如果您未启用 Pipeline Engine，您可以将此变量设置为 CPU 核数的一半。 | 每个 BE 上用于扫描节点的实例数。默认值为 `1`。               |
| pipeline_dop                        | v2.3、v2.4 及 v2.5 | 0                                                            | Pipeline 实例的并行度，用于调整查询并发度。默认值：0，表示系统自动调整每个 Pipeline 实例的并行度。<br />自 v3.0 起，StarRocks 根据查询并行度自适应调整该参数。 |

- 全局设置 `is_report_success` 为 `false`：

  ```SQL
  SET GLOBAL is_report_success = false;
  ```

- 全局设置 `enable_profile` 为 `false`：

  ```SQL
  SET GLOBAL enable_profile = false;
  ```

- 全局设置 `enable_pipeline_engine` 为 `true`：

  ```SQL
  SET GLOBAL enable_pipeline_engine = true;
  ```

- 全局设置 `parallel_fragment_exec_instance_num` 为 `1`：

  ```SQL
  SET GLOBAL parallel_fragment_exec_instance_num = 1;
  ```

- 全局设置 `pipeline_dop` 为 `0`：

  ```SQL
  SET GLOBAL pipeline_dop = 0;
  ```

有关系统变量的更多信息，请参阅 [系统变量](../reference/System_variable.md)。

## 设置用户属性

如果您在集群中创建了新用户，则需要增加新用户的最大连接数（例如至 `1000`）：

```SQL
-- 将 <username> 替换为需要增加最大连接数的用户名。
SET PROPERTY FOR '<username>' 'max_user_connections' = '1000';
```

## 使用 StarRocks 存算分离集群

如果您部署了 [StarRocks 存算分离集群](./deploy_shared_data.md)，则需要在使用时参考当前章节。

StarRocks 存算分离集群的使用类似于 StarRocks 存算一体集群，不同之处在于存算分离集群需要使用存储卷和云原生表才能将数据持久化到 HDFS 或对象存储。

### 创建默认存储卷

您可以使用 StarRocks 自动创建的内置存储卷，也可以手动创建和设置默认存储卷。本节介绍如何手动创建并设置默认存储卷。

> **说明**
>
> 如果您的 StarRocks 存算分离集群是由 v3.0 升级，则无需定义默认存储卷。 StarRocks 会根据您在 FE 配置文件 **fe.conf** 中指定的相关配置项自动创建默认存储卷。您仍然可以使用其他远程数据存储资源创建新的存储卷或定义其他存储卷为默认。

为了确保 StarRocks 存算分离集群有权限在远程数据源中存储数据，您必须在创建数据库或云原生表时引用存储卷。存储卷由远程数据存储系统的属性和凭证信息组成。在部署新 StarRocks 存算分离集群时，如果您禁止了 StarRocks 创建内置存储卷 (将 `enable_load_volume_from_conf` 设置为 `false`)，则启动后必须先创建和设置默认存储卷，然后才能在集群中创建数据库和表。

以下示例使用 IAM user-based 认证为 AWS S3 存储空间 `defaultbucket` 创建存储卷 `def_volume`，激活并将其设置为默认存储卷：

```SQL
CREATE STORAGE VOLUME def_volume
TYPE = S3
LOCATIONS = ("s3://defaultbucket/test/")
PROPERTIES
(
    "enabled" = "true",
    "aws.s3.region" = "us-west-2",
    "aws.s3.endpoint" = "https://s3.us-west-2.amazonaws.com",
    "aws.s3.use_aws_sdk_default_behavior" = "false",
    "aws.s3.use_instance_profile" = "false",
    "aws.s3.access_key" = "xxxxxxxxxx",
    "aws.s3.secret_key" = "yyyyyyyyyy"
);

SET def_volume AS DEFAULT STORAGE VOLUME;
```

有关如何为其他远程存储创建存储卷和设置默认存储卷的更多信息，请参阅 [CREATE STORAGE VOLUME](../sql-reference/sql-statements/Administration/CREATE%20STORAGE%20VOLUME.md) 和 [SET DEFAULT STORAGE VOLUME](../sql-reference/sql-statements/Administration/SET%20DEFAULT%20STORAGE%20VOLUME.md)。

### 创建数据库和云原生表

创建默认存储卷后，您可以使用该存储卷创建数据库和云原生表。

目前，StarRocks 存算分离集群支持以下数据模型：

- 明细模型（Duplicate Key）
- 聚合模型（Aggregate Key）
- 更新模型（Unique Key）
- 主键模型（Primary Key）（当前暂不支持持久化主键索引）

以下示例创建数据库 `cloud_db`，并基于明细模型创建表 `detail_demo`，启用本地磁盘缓存，将热数据有效期设置为一个月，并禁用异步数据导入：

```SQL
CREATE DATABASE cloud_db;
USE cloud_db;
CREATE TABLE IF NOT EXISTS detail_demo (
    recruit_date  DATE           NOT NULL COMMENT "YYYY-MM-DD",
    region_num    TINYINT        COMMENT "range [-128, 127]",
    num_plate     SMALLINT       COMMENT "range [-32768, 32767] ",
    tel           INT            COMMENT "range [-2147483648, 2147483647]",
    id            BIGINT         COMMENT "range [-2^63 + 1 ~ 2^63 - 1]",
    password      LARGEINT       COMMENT "range [-2^127 + 1 ~ 2^127 - 1]",
    name          CHAR(20)       NOT NULL COMMENT "range char(m),m in (1-255) ",
    profile       VARCHAR(500)   NOT NULL COMMENT "upper limit value 65533 bytes",
    ispass        BOOLEAN        COMMENT "true/false")
DUPLICATE KEY(recruit_date, region_num)
DISTRIBUTED BY HASH(recruit_date, region_num)
PROPERTIES (
    "storage_volume" = "def_volume",
    "datacache.enable" = "true",
    "datacache.partition_duration" = "1 MONTH",
    "enable_async_write_back" = "false"
);
```

> **说明**
>
> 当您在 StarRocks 存算分离集群中创建数据库或云原生表时，如果未指定存储卷，StarRocks 将使用默认存储卷。

除了常规表 PROPERTIES 之外，您还需要在创建表时指定以下 PROPERTIES：

| **属性**                | **描述**                                                     |
| ----------------------- | ------------------------------------------------------------ |
| datacache.enable        | 是否启用本地磁盘缓存。默认值：`true`。<ul><li>当该属性设置为 `true` 时，数据会同时导入对象存储（或 HDFS）和本地磁盘（作为查询加速的缓存）。</li><li>当该属性设置为 `false` 时，数据仅导入到对象存储中。</li></ul>**说明**<br />如需启用本地磁盘缓存，必须在 BE 配置项 `storage_root_path` 中指定磁盘目录。 |
| datacache.partition_duration | 热数据的有效期。当启用本地磁盘缓存时，所有数据都会导入至本地磁盘缓存中。当缓存满时，StarRocks 会从缓存中删除最近较少使用（Less recently used）的数据。当有查询需要扫描已删除的数据时，StarRocks 会检查该数据是否在有效期内。如果数据在有效期内，StarRocks 会再次将数据导入至缓存中。如果数据不在有效期内，StarRocks 不会将其导入至缓存中。该属性为字符串，您可以使用以下单位指定：`YEAR`、`MONTH`、`DAY` 和 `HOUR`，例如，`7 DAY` 和 `12 HOUR`。如果不指定，StarRocks 将所有数据都作为热数据进行缓存。<br />**说明**<br />仅当 `datacache.enable` 设置为 `true` 时，此属性可用。 |
| enable_async_write_back | 是否允许数据异步写入对象存储。默认值：`false`。<ul><li>当该属性设置为 `true` 时，导入任务在数据写入本地磁盘缓存后立即返回成功，数据将异步写入对象存储。允许数据异步写入可以提升导入性能，但如果系统发生故障，可能会存在一定的数据可靠性风险。</li><li>当该属性设置为 `false` 时，只有在数据同时写入对象存储和本地磁盘缓存后，导入任务才会返回成功。禁用数据异步写入保证了更高的可用性，但会导致较低的导入性能。</li></ul> |

有关建表的更多信息，请参阅 [CREATE TABLE](../sql-reference/sql-statements/data-definition/CREATE%20TABLE.md)

### 查看表信息

您可以通过 `SHOW PROC "/dbs/<db_id>"` 查看特定数据库中的表的信息。详细信息，请参阅 [SHOW PROC](../sql-reference/sql-statements/Administration/SHOW%20PROC.md)。

示例：

```Plain
mysql> SHOW PROC "/dbs/xxxxx";
+---------+-------------+----------+---------------------+--------------+--------+--------------+--------------------------+--------------+---------------+------------------------------+
| TableId | TableName   | IndexNum | PartitionColumnName | PartitionNum | State  | Type         | LastConsistencyCheckTime | ReplicaCount | PartitionType | StoragePath                  |
+---------+-------------+----------+---------------------+--------------+--------+--------------+--------------------------+--------------+---------------+------------------------------+
| 12003   | detail_demo | 1        | NULL                | 1            | NORMAL | CLOUD_NATIVE | NULL                     | 8            | UNPARTITIONED | s3://xxxxxxxxxxxxxx/1/12003/ |
+---------+-------------+----------+---------------------+--------------+--------+--------------+--------------------------+--------------+---------------+------------------------------+
```

StarRocks 存算分离集群中表的 `Type` 为 `CLOUD_NATIVE`。`StoragePath` 字段为表在对象存储中的路径。

### 向 StarRocks 存算分离集群导入数据

StarRocks 存算分离集群支持 StarRocks 提供的所有导入方式。详细信息，请参阅 [导入总览](../loading/Loading_intro.md)。

### 在 StarRocks 存算分离集群查询

StarRocks 存算分离集群支持 StarRocks 提供的所有查询方式。详细信息，请参阅 [SELECT](../sql-reference/sql-statements/data-manipulation/SELECT.md)。

### StarRocks 存算分离集群限制

StarRocks 存算分离集群暂不支持[同步物化视图](../using_starrocks/Materialized_view-single_table.md)。

## 下一步

成功部署和设置 StarRocks 集群后，您可以开始着手设计最适合您的业务场景场景的表。有关表设计的详细说明，请参阅 [理解表设计](../table_design/StarRocks_table_design.md)。
