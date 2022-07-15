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
- frontend服务将在4566端口监听
- kafka服务将在9092端口监听
- promethus服务将在9500端口监听
- grafana服务将在3000端口监听

**通过psql执行查询语句**
```bash
docker-compose run psql
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
- ad_impression
  ```json
  {
    "bid_id": 1234,
    "ad_id": 1,
    "impression_timestamp": "2022-01-01 19:01:40"
  }
  ```
  其中`bid_id`指某次广告投放的编号，`ad_id`指该次广告的编号，`impression_timestamp`指该次广告投放的时间
- ad_click
  ```json
  {
    "bid_id": 2439384144522347,
    "click_timestamp": "2022-05-23 14:12:56"
  }
  ```
  其中`bid_id`指某次广告投放的编号，`click_timestamp`指该次投放的广告被点击的时间

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
 ad_id |              ctr               
-------+--------------------------------
     1 | 0.7192716236722306525037936267
     2 | 0.7147286821705426356589147287
     3 | 0.7301829268292682926829268293
     4 | 0.6820208023774145616641901932
```
查询每五分钟广告点击率
```sql
dev=> select * from ad_ctr_5min;
 ad_id |              ctr               |     window_end      
-------+--------------------------------+---------------------
     1 |                  0.54003906250 | 2022-07-15 07:40:00
     1 | 0.5550541516245487364620938628 | 2022-07-15 07:45:00
     1 | 0.6104023552502453385672227674 | 2022-07-15 07:50:00
     2 | 0.5356783919597989949748743719 | 2022-07-15 07:40:00
     2 | 0.5117903930131004366812227074 | 2022-07-15 07:45:00
     2 | 0.6036960985626283367556468172 | 2022-07-15 07:50:00
     3 | 0.5359922178988326848249027237 | 2022-07-15 07:40:00
     3 | 0.5317667536988685813751087903 | 2022-07-15 07:45:00
     3 | 0.6038338658146964856230031949 | 2022-07-15 07:50:00
```