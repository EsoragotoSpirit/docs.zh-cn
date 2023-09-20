# 部署 StarRocks 存算分离集群

本文介绍如何部署和使用 StarRocks 存算分离集群。该功能从 3.0 版本开始支持。

## 概述

StarRocks 存算分离集群采用了存储计算分离架构，特别为云存储设计。在存算分离的模式下，StarRocks 将数据存储在兼容 S3 协议的对象存储（例如 AWS S3、OSS 以及 MinIO）或 HDFS 中，而本地盘作为热数据缓存，用以加速查询。通过存储计算分离架构，您可以降低存储成本并且优化资源隔离。除此之外，集群的弹性扩展能力也得以加强。在查询命中缓存的情况下，存算分离集群的查询性能与存算一体集群性能一致。

相对存算一体架构，StarRocks 的存储计算分离架构提供以下优势：

- 廉价且可无缝扩展的存储。
- 弹性可扩展的计算能力。由于数据不再存储在 BE 节点中，因此集群无需进行跨节点数据迁移或 Shuffle 即可完成扩缩容。
- 热数据的本地磁盘缓存，用以提高查询性能。
- 可选异步导入数据至对象存储，提高导入效率。

StarRocks 存算分离集群架构如下：

![Shared-data Architecture](../assets/share_data_arch.png)

## 准备工作

如果您使用 HDFS 作为 StarRocks 存算分离集群的存储，则需要额外做以下准备工作。

### （可选）配置 Kerberos

如果您选择通过 Kerberos 访问 HDFS 存储，则需要根据以下步骤配置 Kerberos 服务：

1. 生成并分发 Keytab 文件（一次性操作）。

   使用 KDC 生成 StarRocks 服务所需的 Keytab 文件，并将其复制到所有 FE 和 BE 节点实例的 **conf** 路径下。

2. 在所有 StarRocks 节点实例上安装 Kerberos Workstation 软件包。

   - CentOS 系统：

   ```Bash
   yum -y install krb5-workstation
   ```

   - Ubuntu 系统：

   ```Bash
   apt install -y krb5-user
   ```

3. 在所有 StarRocks 节点实例上配置 **/etc/krb5.conf** 文件。

   请确保 **/etc/krb5.conf** 中的设置项与需要访问的 HDFS 集群相匹配。需要特别关注的配置项包括 `default_ccache_name`、`realms、ticket_lifetime` 以及 `renew_lifetime`。

4. 创建 **/etc/krb5.conf.d** 路径。

   ```Bash
   mkdir -p /etc/krb5.conf.d
   ```

5. 通过 crontab 定期生成 ccache 文件，ccache 的生成周期需小于其过期时间。

   生成 ccache 文件的命令为：

   ```Bash
   # 将 <path_to_principal> 替换为 Kerberos Principal 的实际路径，
   # 将 <path_to_keytab> 替换为 Keytab 文件的实际路径。
   kinit <path_to_principal> -kt <path_to_keytab>
   ```

   示例

   ```Bash
   0 */8 * * * /usr/bin/kinit -kt /opt/starrocks/conf/starrocks.keytab starrocks@XXX.COM
   ```

6. 复制已开启 Kerberos 认证的 HDFS 集群的 **core-site.xml** 和 **hdfs-site.xml** 文件到所有 StarRocks 节点实例的 **conf** 路径下。

### （可选）配置 HDFS HA 模式

StarRocks 存算分离集群的各个节点会通过 HDFS 客户端访问 HDFS 集群。一般情况下，StarRocks 会按照默认配置来启动 HDFS 客户端，无需手动配置。

但如果您的 HDFS 集群开启了高可用（High Availability，简称为 “HA”）模式，则需要复制 HDFS 集群中的 **hdfs-site.xml** 文件到所有 StarRocks 节点实例的 **conf** 路径下。

## 配置 StarRocks 存算分离集群

StarRocks 存算分离集群的部署方式与存算一体集群的部署方式类似。唯一不同的是 FE 和 BE 的配置文件 **fe.conf** 和 **be.conf** 中的配置项。本小节仅列出部署 StarRocks 存算分离集群时需要添加到配置文件中的 FE 和 BE 配置项。有关部署 StarRocks 存算一体集群的详细说明，请参阅 [部署 StarRocks](/deployment/deploy_manually.md)。

### 配置存算分离集群 FE 节点

在启动 FE 之前，在 FE 配置文件 **fe.conf** 中添加以下配置项：

| **配置项**                          | **描述**                                                     |
| ----------------------------------- | ------------------------------------------------------------ |
| run_mode                            | StarRocks 集群的运行模式。有效值：`shared_data` 和 `shared_nothing` (默认)。`shared_data` 表示在存算分离模式下运行 StarRocks。`shared_nothing` 表示在存算一体模式下运行 StarRocks。<br />**注意**<br />StarRocks 集群不支持存算分离和存算一体模式混合部署。<br />请勿在集群部署完成后更改 `run_mode`，否则将导致集群无法再次启动。不支持从存算一体集群转换为存算分离集群，反之亦然。 |
| cloud_native_meta_port              | 云原生元数据服务监听端口。默认值：`6090`。                   |
| enable_load_volume_from_conf | 是否允许 StarRocks 使用 FE 配置文件中指定的存储相关属性创建默认存储卷。有效值：`true`（默认）和 `false`。自 v3.1.0 起支持。<ul><li>如果您在创建新的存算分离集群时指定此项为 `true`，StarRocks 将使用 FE 配置文件中存储相关属性创建内置存储卷 `builtin_storage_volume`，并将其设置为默认存储卷。但如果您没有指定存储相关的属性，StarRocks 将无法启动。</li><li>如果您在创建新的存算分离集群时指定此项为 `false`，StarRocks 将直接启动，不会创建内置存储卷。在 StarRocks 中创建任何对象之前，您必须手动创建一个存储卷并将其设置为默认存储卷。详细信息请参见[创建默认存储卷](./post_deployment_setup.md#创建默认存储卷)。</li></ul>**注意**<br />建议您在升级现有的 v3.0 存算分离集群时，保留此项的默认配置 `true`。如果将此项修改为 `false`，升级前创建的数据库和表将变为只读，您无法向其中导入数据。 |
| cloud_native_storage_type           | 您使用的存储类型。在存算分离模式下，StarRocks 支持将数据存储在 HDFS 、Azure Blob（自 v3.1.1 起支持）、以及兼容 S3 协议的对象存储中（例如 AWS S3、Google GCP、阿里云 OSS 以及 MinIO）。有效值：`S3`（默认）、`AZBLOB` 和 `HDFS`。如果您将此项指定为 `S3`，则必须添加以 `aws_s3` 为前缀的配置项。如果您将此项指定为 `AZBLOB`，则必须添加以 `azure_blob` 为前缀的配置项。如果将此项指定为 `HDFS`，则只需指定 `cloud_native_hdfs_url`。 |
| cloud_native_hdfs_url               | HDFS 存储的 URL，例如 `hdfs://127.0.0.1:9000/user/xxx/starrocks/`。 |
| aws_s3_path                         | 用于存储数据的 S3 存储空间路径，由 S3 存储桶的名称及其下的子路径（如有）组成，如 `testbucket/subpath`。 |
| aws_s3_region                       | 需访问的 S3 存储空间的地区，如 `us-west-2`。                 |
| aws_s3_endpoint                     | 访问 S3 存储空间的连接地址，如 `https://s3.us-west-2.amazonaws.com`。 |
| aws_s3_use_aws_sdk_default_behavior | 是否使用 AWS SDK 默认的认证凭证。有效值：`true` 和 `false` (默认)。 |
| aws_s3_use_instance_profile         | 是否使用 Instance Profile 或 Assumed Role 作为安全凭证访问 S3。有效值：`true` 和 `false` (默认)。<ul><li>如果您使用 IAM 用户凭证（Access Key 和 Secret Key）访问 S3，则需要将此项设为 `false`，并指定 `aws_s3_access_key` 和 `aws_s3_secret_key`。</li><li>如果您使用 Instance Profile 访问 S3，则需要将此项设为 `true`。</li><li>如果您使用 Assumed Role 访问 S3，则需要将此项设为 `true`，并指定 `aws_s3_iam_role_arn`。</li><li>如果您使用外部 AWS 账户通过 Assumed Role 认证访问 S3，则需要额外指定 `aws_s3_external_id`。</li></ul> |
| aws_s3_access_key                   | 访问 S3 存储空间的 Access Key。                              |
| aws_s3_secret_key                   | 访问 S3 存储空间的 Secret Key。                              |
| aws_s3_iam_role_arn                 | 有访问 S3 存储空间权限 IAM Role 的 ARN。                      |
| aws_s3_external_id                  | 用于跨 AWS 账户访问 S3 存储空间的外部 ID。                     |
| azure_blob_path                     | 用于存储数据的 Azure Blob Storage 路径，由 Storage Account 中的容器名称和容器下的子路径（如有）组成，如 `testcontainer/subpath`。 |
| azure_blob_endpoint                 | Azure Blob Storage 的链接地址，如 `https://test.blob.core.windows.net`。 |
| azure_blob_shared_key               | 访问 Azure Blob Storage 的 Shared Key。                     |
| azure_blob_sas_token                | 访问 Azure Blob Storage 的共享访问签名（SAS）。                |

> **注意**
>
> 成功创建存算分离集群后，您只能修改与安全凭证相关的配置项。如果您更改了原有存储路径相关的配置项，则在此之前创建的数据库和表将变为只读，您无法向其中导入数据。

如果您想在集群创建后手动创建默认存储卷，则只需添加以下配置项：

```Properties
run_mode = shared_data
cloud_native_meta_port = <meta_port>
enable_load_volume_from_conf = false
```

如果您想在 FE 配置文件中指定存储相关的属性，示例如下：

- 如果您使用 HDFS 存储，请添加以下配置项：

  ```Properties
  run_mode = shared_data
  cloud_native_meta_port = <meta_port>
  cloud_native_storage_type = HDFS
  cloud_native_hdfs_url = <hdfs_url>
  ```

- 如果您使用 AWS S3：

  - 如果您使用 AWS SDK 默认的认证凭证，请添加以下配置项：

    ```Properties
    run_mode = shared_data
    cloud_native_meta_port = <meta_port>
    cloud_native_storage_type = S3

    # 如 testbucket/subpath
    aws_s3_path = <s3_path>

    # 如 us-west-2
    aws_s3_region = <region>

    # 如 https://s3.us-west-2.amazonaws.com
    aws_s3_endpoint = <endpoint_url>

    aws_s3_use_aws_sdk_default_behavior = true
    ```

  - 如果您使用 IAM user-based 认证，请添加以下配置项：

    ```Properties
    run_mode = shared_data
    cloud_native_meta_port = <meta_port>
    cloud_native_storage_type = S3
    aws_s3_path = <s3_path>
    aws_s3_region = <region>
    aws_s3_endpoint = <endpoint_url>
    aws_s3_access_key = <access_key>
    aws_s3_secret_key = <secret_key>
    ```

  - 如果您使用 Instance Profile 认证，请添加以下配置项：

    ```Properties
    run_mode = shared_data
    cloud_native_meta_port = <meta_port>
    cloud_native_storage_type = S3

    # 如 testbucket/subpath
    aws_s3_path = <s3_path>

    # 如 us-west-2
    aws_s3_region = <region>

    # 如 https://s3.us-west-2.amazonaws.com
    aws_s3_endpoint = <endpoint_url>

    aws_s3_use_instance_profile = true
    ```

  - 如果您使用 Assumed Role 认证，请添加以下配置项：

    ```Properties
    run_mode = shared_data
    cloud_native_meta_port = <meta_port>
    cloud_native_storage_type = S3

    # 如 testbucket/subpath
    aws_s3_path = <s3_path>

    # 如 us-west-2
    aws_s3_region = <region>

    # 如 https://s3.us-west-2.amazonaws.com
    aws_s3_endpoint = <endpoint_url>

    aws_s3_use_instance_profile = true
    aws_s3_iam_role_arn = <role_arn>
    ```

  - 如果您使用外部 AWS 账户通过 Assumed Role 认证，请添加以下配置项：

    ```Properties
    run_mode = shared_data
    cloud_native_meta_port = <meta_port>
    cloud_native_storage_type = S3

    # 如 testbucket/subpath
    aws_s3_path = <s3_path>

    # 如 us-west-2
    aws_s3_region = <region>

    # 如 https://s3.us-west-2.amazonaws.com
    aws_s3_endpoint = <endpoint_url>

    aws_s3_use_instance_profile = true
    aws_s3_iam_role_arn = <role_arn>
    aws_s3_external_id = <external_id>
    ```

- 如果您使用 Azure Blob Storage（自 v3.1.1 起支持）：

  - 如果您使用共享密钥（Shared Key）认证，请添加以下配置项：

    ```Properties
    run_mode = shared_data
    cloud_native_meta_port = <meta_port>
    cloud_native_storage_type = AZBLOB

    # 如 testcontainer/subpath
    azure_blob_path = <blob_path>

    # 如 https://test.blob.core.windows.net
    azure_blob_endpoint = <endpoint_url>

    azure_blob_shared_key = <shared_key>
    ```

  - 如果您使用共享访问签名（SAS）认证，请添加以下配置项：

    ```Properties
    run_mode = shared_data
    cloud_native_meta_port = <meta_port>
    cloud_native_storage_type = AZBLOB

    # 如 testcontainer/subpath
    azure_blob_path = <blob_path>

    # 如 https://test.blob.core.windows.net
    azure_blob_endpoint = <endpoint_url>

    azure_blob_sas_token = <sas_token>
    ```

  > **注意**
  >
  > 创建 Azure Blob Storage Account 时必须禁用分层命名空间。

- 如果您使用 GCP Cloud Storage：

  ```Properties
  run_mode = shared_data
  cloud_native_meta_port = <meta_port>
  cloud_native_storage_type = S3

  # 如 testbucket/subpath
  aws_s3_path = <s3_path>

  # 例如：us-east-1
  aws_s3_region = <region>

  # 例如：https://storage.googleapis.com
  aws_s3_endpoint = <endpoint_url>

  aws_s3_access_key = <access_key>
  aws_s3_secret_key = <secret_key>
  ```

- 如果您使用阿里云 OSS：

  ```Properties
  run_mode = shared_data
  cloud_native_meta_port = <meta_port>
  cloud_native_storage_type = S3

  # 如 testbucket/subpath
  aws_s3_path = <s3_path>

  # 例如：cn-zhangjiakou
  aws_s3_region = <region>

  # 例如：https://oss-cn-zhangjiakou-internal.aliyuncs.com
  aws_s3_endpoint = <endpoint_url>

  aws_s3_access_key = <access_key>
  aws_s3_secret_key = <secret_key>
  ```

- 如果您使用华为云 OBS：

  ```Properties
  run_mode = shared_data
  cloud_native_meta_port = <meta_port>
  cloud_native_storage_type = S3

  # 如 testbucket/subpath
  aws_s3_path = <s3_path>

  # 例如：cn-north-4
  aws_s3_region = <region>

  # 例如：https://obs.cn-north-4.myhuaweicloud.com
  aws_s3_endpoint = <endpoint_url>

  aws_s3_access_key = <access_key>
  aws_s3_secret_key = <secret_key>
  ```

- 如果您使用腾讯云 COS：

  ```Properties
  run_mode = shared_data
  cloud_native_meta_port = <meta_port>
  cloud_native_storage_type = S3

  # 如 testbucket/subpath
  aws_s3_path = <s3_path>

  # 例如：ap-beijing
  aws_s3_region = <region>

  # 例如：https://cos.ap-beijing.myqcloud.com
  aws_s3_endpoint = <endpoint_url>

  aws_s3_access_key = <access_key>
  aws_s3_secret_key = <secret_key>
  ```

- 如果您使用火山引擎 TOS：

  ```Properties
  run_mode = shared_data
  cloud_native_meta_port = <meta_port>
  cloud_native_storage_type = S3

  # 如 testbucket/subpath
  aws_s3_path = <s3_path>

  # 例如：cn-beijing
  aws_s3_region = <region>

  # 例如：https://tos-s3-cn-beijing.ivolces.com
  aws_s3_endpoint = <endpoint_url>

  aws_s3_access_key = <access_key>
  aws_s3_secret_key = <secret_key>
  ```

- 如果您使用金山云：

  ```Properties
  run_mode = shared_data
  cloud_native_meta_port = <meta_port>
  cloud_native_storage_type = S3

  # 如 testbucket/subpath
  aws_s3_path = <s3_path>

  # 例如：BEIJING
  aws_s3_region = <region>
  
  # 注意请使用三级域名, 金山云不支持二级域名
  # 例如：jeff-test.ks3-cn-beijing.ksyuncs.com
  aws_s3_endpoint = <endpoint_url>

  aws_s3_access_key = <access_key>
  aws_s3_secret_key = <secret_key>
  ```

- 如果您使用 MinIO：

  ```Properties
  run_mode = shared_data
  cloud_native_meta_port = <meta_port>
  cloud_native_storage_type = S3

  # 如 testbucket/subpath
  aws_s3_path = <s3_path>

  # 例如：us-east-1
  aws_s3_region = <region>

  # 例如：http://172.26.xx.xxx:39000
  aws_s3_endpoint = <endpoint_url>

  aws_s3_access_key = <access_key>
  aws_s3_secret_key = <secret_key>
  ```

- 如果您使用 Ceph S3：

  ```Properties
  run_mode = shared_data
  cloud_native_meta_port = <meta_port>
  cloud_native_storage_type = S3

  # 如 testbucket/subpath
  aws_s3_path = <s3_path>
  
  # 例如：http://172.26.xx.xxx:7480
  aws_s3_endpoint = <endpoint_url>

  aws_s3_access_key = <access_key>
  aws_s3_secret_key = <secret_key>
  ```

### 配置存算分离集群 BE 节点

**在启动 BE 之前**，在 BE 配置文件 **be.conf** 中添加以下配置项：

```Properties
starlet_port = <starlet_port>
storage_root_path = <storage_root_path>
```

| **配置项**              | **描述**                 |
| ---------------------- | ------------------------ |
| starlet_port | 存算分离模式下，用于 BE 心跳服务的端口。默认值：`9070`。 |
| storage_root_path      | 本地缓存数据依赖的存储目录以及该存储介质的类型，多块盘配置使用分号（;）隔开。如果为 SSD 磁盘，需在路径后添加 `,medium:ssd`，如果为 HDD 磁盘，需在路径后添加 `,medium:hdd`。例如：`/data1,medium:hdd;/data2,medium:ssd`。默认值：`${STARROCKS_HOME}/storage`。 |

> **说明**
>
> 本地缓存数据将存储在 **<storage_root_path\>/starlet_cache** 路径下。

## 下一步

- 关于如何启动 StarRocks 集群各节点，请参考[部署 StarRocks](/deployment/deploy_manually.md)。
- 启动 StarRocks 存算分离集群后，您可以进一步了解[存算分离集群的使用](./post_deployment_setup.md#关于使用-starrocks-存算分离集群)。
