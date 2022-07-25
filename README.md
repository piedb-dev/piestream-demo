# Piestream 演示说明

## 基本使用

**拉取所需镜像**
```bash
docker-compose pull
```

**启动 piestream**
```bash
docker-compose up -d
```
该脚本将启动如下服务
- meta x 1
- compute x 2
- frontend x 1
- compactor x 1
- etcd x 1
- minio x 1
- promethus x 1
- grafana x 1

若启动无误，
- frontend服务将在5373端口监听
- promethus服务将在9500端口监听
- grafana服务将在3000端口监听

**连接 piestream**
```bash
docker-compose run psql
```

**创建视图**
```sql
-- 基本语法
CREATE [MATERIALIZED] SOURCE [IF NOT EXISTS] source_name (
   column_name data_type,...
)
WITH (
   connector='kafka',
   field_name='value', ...
)
ROW FORMAT JSON;
-- 示例
CREATE MATERIALIZED SOURCE IF NOT EXISTS my_source (
   field1 varchar,
   field2 bigint,
)
WITH (
   connector='kafka',
   kafka.topic='test_topic',
   kafka.brokers='10.0.0.1:9092,10.0.0.2:9092',
   kafka.scan.startup.mode='latest'
)
ROW FORMAT JSON;
```
其中`kafka.scan.startup.mode`用于指定 piestream 应当从消息队列头部还是尾部开始扫描，默认取值为`earliest`，其他可选取值为`latest`。

**创建视图**
```sql
-- 基本语法
CREATE MATERIALIZED VIEW <mv> AS <select_query>;
-- 示例
CREATE MATERIALIZED VIEW last_mv AS SELECT * FROM my_source ORDER BY id DESC LIMIT 100000;
```

**查询视图**
```sql
SELECT * FROM last_mv LIMIT 5;
```

**停止所有服务并清空相关磁盘占用**
```bash
docker-compose down -v --remove-orphans
```

## 广告点击率案例

**拉取额外镜像**
```bash
docker-compose --profile demo pull
```

**启动 piestream 并加载演示数据**
```bash
docker-compose --profile demo up -d
```
除启动各项服务外，该命令将额外启动loadgen服务，并向kafka发送随机演示数据。具体来说，该服务将向kafka发送以下两个topic的数据：
- ad_exposure
  ```json
  {
    "advertise_id": 1234,
    "vendor_id": 1,
    "exposed_at": "2022-01-01 19:01:40"
  }
  ```
  其中`advertise_id`指某次广告投放的编号，`vendor_id`指该广告的编号，`impression_timestamp`指该次广告投放的时间
- ad_click
  ```json
  {
    "advertise_id": 2439384144522347,
    "clicked_at": "2022-05-23 14:12:56"
  }
  ```
  其中`advertise_id`指某次广告投放的编号，`clicked_at`指该次投放的广告被点击的时间

**加载建视图语句**  
相关建视图语句在`rise.sql`文件中
```bash
$ docker-compose run load-view
CREATE_SOURCE
CREATE_SOURCE
CREATE_MATERIALIZED_VIEW
CREATE_MATERIALIZED_VIEW
```
**通过psql执行查询语句**
```bash
docker-compose run psql
```

查询广告点击率
```sql
dev=> select * from ad_ctr;
 vendor_id |              ctr               
-----------+--------------------------------
         1 | 0.6696035242290748898678414097
         2 | 0.7277486910994764397905759162
         3 | 0.6635514018691588785046728972
         4 | 0.7070707070707070707070707071
         5 | 0.7291666666666666666666666667
```
查询每五分钟广告点击率
```sql
dev=> select * from ad_ctr_5min;
 vendor_id |              ctr               |     window_end      
-----------+--------------------------------+---------------------
         1 | 0.3018433179723502304147465438 | 2022-07-15 12:30:00
         2 | 0.3128342245989304812834224599 | 2022-07-15 12:30:00
         3 | 0.3025641025641025641025641026 | 2022-07-15 12:30:00
         4 | 0.2803030303030303030303030303 | 2022-07-15 12:30:00
         5 | 0.3058035714285714285714285714 | 2022-07-15 12:30:00
```