# DROP STORAGE VOLUME

## 功能

删除指定存储卷。已删除的存储卷无法被引用。该功能自 v3.1 起支持。

> **注意**
>
> - 仅拥有指定存储卷 DROP 权限的用户可以执行该操作。
> - 默认存储卷以及内置存储卷 `builtin_storage_volume` 无法删除。您可以通过 [DESC STORAGE VOLUME](./DESC%20STORAGE%20VOLUME.md) 查看存储卷是否为默认存储卷。
> - 被已有数据库或云原生表引用的存储卷无法删除。

## 语法

```SQL
DROP STORAGE VOLUME [ IF EXISTS ] <storage_volume_name>
```

## 参数说明

| **参数**            | **说明**               |
| ------------------- | ---------------------- |
| storage_volume_name | 待删除的存储卷的名称。 |

## 示例

示例一：删除存储卷 `my_s3_volume`。

```Plain
DROP STORAGE VOLUME my_s3_volume;
```

## 相关 SQL

- [CREATE STORAGE VOLUME](./CREATE%20STORAGE%20VOLUME.md)
- [ALTER STORAGE VOLUME](./ALTER%20STORAGE%20VOLUME.md)
- [SET DEFAULT STORAGE VOLUME](./SET%20DEFAULT%20STORAGE%20VOLUME.md)
- [DESC STORAGE VOLUME](./DESC%20STORAGE%20VOLUME.md)
- [SHOW STORAGE VOLUMES](./SHOW%20STORAGE%20VOLUMES.md)
