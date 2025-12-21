
# 功能简介

作业timeline store的一种基于leveldb的实现。主要保存下面信息：
- start time信息，保存在starttime-ldb里面，是一个单独的LevelDB。
- entity信息，保存在entity-ldb里面，是一个单独的LevelDB，支持按照时间进行归档以及清理。
- indexed entity信息，保存在indexes-ldb里面，是一个单独的LevelDB。支持按照时间进行归档以及清理。
- domain信息，保存domain信息，保存在domain-ldb里面，是一个单独的LevelDB。
- owner信息，保存owner信息，保存在owner-ldb里面，是一个单独的LevelDB。


# 数据库详解


## starttime-ldb
starttime-ldb主要保存的是app的启动时间，保存的信息主要如下：
- 保存的key是EntityIdentifier，value是作业启动时间。
- 当前版本信息，key是timeline-store-version，value是版本信息。

## entity-ldb
entity-ldb数据库的类型是RollingLevelDB，支持按照时间创建单独的数据库。实际上是多个LevelDB数据库，只是LevelDB数据库的名称上面带了时间。方便按照时间获取对应的数据库。

在保存数据到entity-ldb里面，首先需要获取以及保存starttime。

核心代码参考：
```java
Long startTime = getAndSetStartTime(entity.getEntityId(),
          entity.getEntityType(), entity.getStartTime(), events);
```

entity-ldb里面保存和如下信息：
- ENTITY_ENTRY_PREFIX + entity type + revstarttime + entity id
- ENTITY_ENTRY_PREFIX + entity type + revstarttime + entity id + DOMAIN_ID_COLUMN
- ENTITY_ENTRY_PREFIX + entity type + revstarttime + entity id + EVENTS_COLUMN + reveventtimestamp + eventtype
- ENTITY_ENTRY_PREFIX + entity type + revstarttime + entity id + PRIMARY_FILTERS_COLUMN + name + value
- ENTITY_ENTRY_PREFIX + entity type + revstarttime + entity id + OTHER_INFO_COLUMN + name
- ENTITY_ENTRY_PREFIX + entity type + revstarttime + entity id + RELATED_ENTITIES_COLUMN + relatedentity type + relatedentity id

关键字段含义
| 字段 |  含义| 类型 | 
| ----|---|----|
| DOMAIN_ID_COLUMN        | "d".getBytes(UTF_8)    | byte[] 
| EVENTS_COLUMN           | "e".getBytes(UTF_8)    | byte[]
| PRIMARY_FILTERS_COLUMN  | "f".getBytes(UTF_8)    | byte[]
| OTHER_INFO_COLUMN       | "i".getBytes(UTF_8)    | byte[]
| RELATED_ENTITIES_COLUMN | "r".getBytes(UTF_8)    | byte[]
| ENTITY_ENTRY_PREFIX     | 3个空字符               | ---
| revstarttime            | 启动时间，由低八位存储   | byte[]



## indexes-ldb

主要是entity的索引信息，key的格式如下：INDEXED_ENTRY_PREFIX + primaryfilter name + primaryfilter value +  key


## domain-ldb

主要保存domain信息，根据作业类型的不同而不同，主要是Tez任务保存的比较多，带了作业的ID，当前数据库没有清理，可能会造成数据残留，详见：[YARN-11911](https://issues.apache.org/jira/browse/YARN-11911)

保存的信息如下：
| 信息 | key | 备注 |
| ----|----|----|
| 描述信息      | domainId + DESCRIPTION_COLUMN   | DESCRIPTION_COLUMN ="d".getBytes(UTF_8)
| owner 信息   | domainId + OWNER_COLUMN          | OWNER_COLUMN = "o".getBytes(UTF_8)
| reader信息   | domainId + READER_COLUMN         | READER_COLUMN = "r".getBytes(UTF_8)
| writer信息   | domainId + WRITER_COLUMN         | "w".getBytes(UTF_8)
| 时间信息     | domainId + TIMESTAMP_COLUMN       | TIMESTAMP_COLUMN = "t".getBytes(UTF_8),低八位为创建时间，高八位为修改时间


## owner-ldb

主要保存owner信息，根据作业类型的不同而不同，主要是Tez任务保存的比较多，带了作业的ID，当前数据库没有清理，可能会造成数据残留，详见：[YARN-11911](https://issues.apache.org/jira/browse/YARN-11911)

保存的信息如下：
| 信息 | key | 备注 |
| ----|----|----|
| 描述信息      | owner + domainId + DESCRIPTION_COLUMN    | DESCRIPTION_COLUMN ="d".getBytes(UTF_8)
| owner 信息    | owner + domainId + OWNER_COLUMN          | OWNER_COLUMN = "o".getBytes(UTF_8)
| reader信息   | owner + domainId + READER_COLUMN          | READER_COLUMN = "r".getBytes(UTF_8)
| writer信息   | owner + domainId + WRITER_COLUMN          | "w".getBytes(UTF_8)
| 时间信息     | owner + domainId + TIMESTAMP_COLUMN       | TIMESTAMP_COLUMN = "t".getBytes(UTF_8),低八位为创建时间，高八位为修改时间

