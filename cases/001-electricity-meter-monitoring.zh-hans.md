# 应用场景 - 电表监控

[English](001-electricity-meter-monitoring.md) | 简体中文

# 建模
电表监控是典型的物联网场景，对于同一类设备，其采集的数据都是规则的、时序的、结构化的，按照设定的周期或受外界触发时进行采集。TDengine 采用关系型数据模型，数据建模需要建库、建表，之后才能插入或查询数据。
## 设计理念
首先针对建库， TDengine 建议将不同数据特征的表创建在不同的库里，因为每个库可以配置不同的存储策略，比如数据保留时长、副本数目、数据块大小、时间精度、是否允许更新数据等等。

建完库后，如果采用传统的方式，将多个设备的数据写入到一张表中，由于网络延时、写入锁保护，不同设备数据到达服务器的时序无法保证并且一个设备的数据难以连续存储在一起，所以可采用一个数据采集点一张表的方式，最大程度保证单个数据采集点的插入、查询性能最优。建议用数据采集点全局唯一 ID 「如设备序列号」来做表名，每个数据采集点可以同时采集多个物理量。

由于采用一个数据采集点一张表，表的数量会剧增，难以管理、聚合，TDengine 引入了超级表（ Stable ）的概念。超级表是某一特定类型的数据采集点的集合，对同一类型数据采集点，其表的结构完全相同，但每个表「每个数据采集点」的静态属性「标签」是不一样的，标签可以有多个，并且后续能增删改。如果整个系统有 N 个不同类型的数据采集点，那就要建立 N 个超级表。
## 表结构
对于电表监控系统，假设一个智能电表同时采集电流（ current ）、电压（ voltage ）、相位（ phase ）这三个量，再加上每条记录都有的设备 ID 、时间戳以及静态标签「如位置、分组」，采集的数据记录类似如下表格：

|  设备ID   | 时间戳  | 电流  | 电压  | 相位 | 位置  | 分组  |
|  ----  | ----  |----  |----  |----  |----  |----  |
| d1001  | 1538548685000 | 10.3 | 219 | 0.31 | Beijing.Chaoyang	 | 2 |
| d1002  | 1538548684000 | 10.2 | 220 | 0.23 | Beijing.Chaoyang	 | 3 |
| d1003| 1538548686500 | 11.5 | 221 | 0.35 | Beijing.Haidian | 3 |
| d1001 | 1538548695000 | 12.6 | 218 | 0.33 | Beijing.Chaoyang | 2 |
| d1002 | 1538548696650 | 10.3 | 218 | 0.25 | Beijing.Chaoyang	 | 3 |

## 建模语句
* 建库。此处将创建一个名为 power 的库，这个库的数据将保留 365 天「超过 365 天将被自动删除」，每 10 天一个数据文件，内存块数为 6，允许更新数据，缓存最新一条记录：

```
CREATE DATABASE power KEEP 365 DAYS 10 BLOCKS 6 UPDATE 2 CACHELAST 1;
```

* 建超级表。与创建普通表类似，创建表时，需要提供表名「此处为 meters 」、数据列的定义。第一列必须为时间戳「此处为 ts 」，其他列为采集的物理量，除此之外，还需要提供标签列的定义 「此处为 location , groupId 」：

```
USE power;
CREATE STABLE meters (ts timestamp, current float, voltage int, phase float) TAGS (location binary(64), groupId int);
```

* 建子表。TDengine 对每个数据采集点需要独立建表，创建时，需要使用超级表做模板，同时指定标签的具体值。后续会介绍在写入数据时使用自动建表语句来创建不存在的表：

```
CREATE TABLE d1001 USING meters TAGS ("Beijing.Chaoyang", 2);
```


# 写入与查询
TDengine 语句丰富，此处仅对写入、查询做简单示例，详情请查看官方文档：[数据写入](https://www.taosdata.com/docs/cn/v2.0/taos-sql#insert)	、 [数据查询](https://www.taosdata.com/docs/cn/v2.0/taos-sql#select) 。

若想直接获取电表监控的测试数据集，可以使用官方压测工具 [taosBenchmark](https://www.taosdata.com/docs/cn/v2.0/getting-started/taosdemo) 生成测试数据，默认数据库名为 test 。
## 数据写入

* 向一个表写入一条或多条记录：

```
INSERT INTO d1001 VALUES (NOW, 10.2, 219, 0.32);

INSERT INTO d1001 VALUES ('2021-07-13 14:06:32.272', 10.2, 219, 0.32) (1626164208000, 10.15, 217, 0.33);

```

* 写入记录，数据对应到指定的列：

```
INSERT INTO d1001 (ts, current, phase) VALUES ('2021-07-13 14:06:33.196', 10.27, 0.31);
```

* 向多个表写入记录：

```
INSERT INTO d1001 VALUES ('2021-07-13 14:06:34.630', 10.2, 219, 0.32) ('2021-07-13 14:06:35.779', 10.15, 217, 0.33)
            d1002 (ts, current, phase) VALUES ('2021-07-13 14:06:34.255', 10.27, 0.31）;
```

* 写入记录时自动建表：

```
INSERT INTO d21001 USING meters TAGS ('Beijing.Chaoyang', 2) VALUES ('2021-07-13 14:06:32.272', 10.2, 219, 0.32);
```

* 导入来自文件的数据记录，csv 文件不需要表头：

```
INSERT INTO d1001 FILE '/tmp/csvfile.csv';
```


## 业务查询

* 对子表、超级表进行全量查询，使用通配符 * 查询超级表会额外展示标签列：

```
SELECT * FROM d1001;
SELECT * FROM meters;
```

* 关联查询，若有带表的通配符，则只返回该表的列数据。普通表之间的 JOIN 操作，只能使用主键时间戳来关联，超级表之间的 JOIN ，还可以使用对应的标签列：

```
SELECT d1001.* FROM d1001, d1003 WHERE d1001.ts=d1003.ts;
```

* 函数查询，count(*) 函数只返回一列，first 、last 、last_row 函数返回全部列：

```
SELECT count(*) FROM d1001;
SELECT first(*) FROM d1001;
```


* 查询超级表中所有子表的最新一条记录：

```
SELECT last_row(*) FROM meters GROUP BY tbname;
```

* 标签列或普通列去重查询，支持输出不重复的组合：

```
SELECT DISTINCT location , groupId  FROM meters;
SELECT DISTINCT current , voltage , phase  FROM d1001;
```

# 数据展示
TDengine 服务启动后会自动创建一个监测数据库 log ，包含对服务器的一系列监控数据，通过 Grafana 和 TDengine 数据源插件，构成了数据库监控、可视化解决方案 TDinsight 。

详细介绍与部署操作请移步：[TDinsight - 基于Grafana的TDengine零依赖监控解决方案](https://www.taosdata.com/docs/cn/v2.0/tools/insight)

