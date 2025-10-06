# . 安装 PostGreSQL & SQLServer &Doris-2.1.6V 数据库 (linux)
# 分别创建 2 张表, 使⽤ FlinkCDC 读取这两张表的数据, 并写⼊下游的 Kafka 主题中, 使⽤ doris 的 Roadtime Load ⽅式
# 进⾏数据加载, 使⽤动态分区，使⽤分桶
#本项目实现从 PostgreSQL 和 SQLServer 数据库通过 Flink CDC 实时同步数据到 Kafka，并使用 Doris Routine Load 方式加载到 Doris 数据仓库，采用动态分区和分桶优化。

系统架构:
PostgreSQL/SQLServer → Flink CDC → Kafka → Doris Routine Load → Doris

项目结构:
stream-realtime/
    ├── src/
    │   └── main/
    │       └── java/
    │           └── com/
    │               └── flink/
    │                   ├── FlinkDoris.java
    │                   └── 
    ├── pom.xml
    └── pg sqlserver doris.md

1. 完成 Doris 的安装与启动
   首先需要完成 Doris 的配置并启动服务：
   bash
# 进入Doris目录（假设你下载的是tar包并解压）
cd apache-doris-2.1.6-bin-x86_64

# 配置FE (Frontend)
cd fe/conf
vim fe.conf
# 设置priority_networks，例如: priority_networks = 192.168.1.0/24
# 保存退出

# 启动FE
cd ../bin
./start_fe.sh --daemon

# 配置BE (Backend)
cd /path/to/doris/be/conf
vim be.conf
# 同样设置priority_networks
# 保存退出

# 启动BE
cd ../bin
./start_be.sh --daemon

# 关联BE到FE（需要MySQL客户端）
mysql -h 127.0.0.1 -P 9030 -u root
ALTER SYSTEM ADD BACKEND "你的服务器IP:9050";
2. 创建数据库表
   2.1 在 PostgreSQL 中创建表
   在 IDEA 的 PostgreSQL 连接中执行以下 SQL：
   PostgreSQL表创建SQL

-- 创建数据库（如果尚未创建）
CREATE DATABASE cdc_demo;

-- 切换到cdc_demo数据库
\c cdc_demo;

-- 创建用户表
CREATE TABLE users (
id SERIAL PRIMARY KEY,
name VARCHAR(50) NOT NULL,
email VARCHAR(100) UNIQUE NOT NULL,
created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
age INT
);

-- 创建订单表
CREATE TABLE orders (
order_id SERIAL PRIMARY KEY,
user_id INT REFERENCES users(id),
product_name VARCHAR(100) NOT NULL,
amount DECIMAL(10, 2) NOT NULL,
order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
status VARCHAR(20)
);

-- 启用CDC所需的扩展
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS wal2json;


   2.2 在 SQL Server 中创建表
   在 IDEA 的 SQL Server 连接中执行以下 SQL：
   SQL Server表创建SQL

-- 创建数据库
CREATE DATABASE cdc_demo;
GO

USE cdc_demo;
GO

-- 启用CDC功能
EXEC sp_cdc_enable_db;
GO

-- 创建产品表
CREATE TABLE products (
product_id INT IDENTITY(1,1) PRIMARY KEY,
product_name VARCHAR(100) NOT NULL,
price DECIMAL(10, 2) NOT NULL,
stock INT NOT NULL,
updated_at DATETIME DEFAULT GETDATE()
);
GO

-- 创建客户表
CREATE TABLE customers (
customer_id INT IDENTITY(1,1) PRIMARY KEY,
first_name VARCHAR(50) NOT NULL,
last_name VARCHAR(50) NOT NULL,
phone VARCHAR(20),
address VARCHAR(255),
registration_date DATETIME DEFAULT GETDATE()
);
GO

-- 为表启用CDC
EXEC sp_cdc_enable_table
@source_schema = 'dbo',
@source_name = 'products',
@role_name = NULL,
@supports_net_changes = 1;
GO

EXEC sp_cdc_enable_table
@source_schema = 'dbo',
@source_name = 'customers',
@role_name = NULL,
@supports_net_changes = 1;
GO



   2.3 在 Doris 中创建目标表
   通过 MySQL 客户端连接 Doris 并执行以下 SQL：
   Doris目标表创建SQL

-- 连接Doris
-- mysql -h 127.0.0.1 -P 9030 -u root

-- 创建数据库
CREATE DATABASE cdc_demo;
USE cdc_demo;

-- 创建用户订单合并表（动态分区+分桶）
CREATE TABLE user_orders (
dt DATE NOT NULL COMMENT "分区日期",
id INT NOT NULL COMMENT "用户ID",
name VARCHAR(50) COMMENT "用户姓名",
order_id INT NOT NULL COMMENT "订单ID",
product_name VARCHAR(100) COMMENT "产品名称",
amount DECIMAL(10, 2) COMMENT "订单金额",
order_date DATETIME COMMENT "订单日期"
) ENGINE=OLAP
DUPLICATE KEY(id, order_id)
PARTITION BY RANGE (dt) ()
DISTRIBUTED BY HASH(id) BUCKETS 8
PROPERTIES (
"dynamic_partition.enable" = "true",
"dynamic_partition.time_unit" = "DAY",
"dynamic_partition.start" = "-30",
"dynamic_partition.end" = "7",
"dynamic_partition.prefix" = "p",
"dynamic_partition.buckets" = "8"
);

-- 创建产品客户合并表（动态分区+分桶）
CREATE TABLE product_customers (
dt DATE NOT NULL COMMENT "分区日期",
product_id INT NOT NULL COMMENT "产品ID",
product_name VARCHAR(100) COMMENT "产品名称",
price DECIMAL(10, 2) COMMENT "产品价格",
customer_id INT NOT NULL COMMENT "客户ID",
customer_name VARCHAR(100) COMMENT "客户姓名",
registration_date DATETIME COMMENT "注册日期"
) ENGINE=OLAP
DUPLICATE KEY(product_id, customer_id)
PARTITION BY RANGE (dt) ()
DISTRIBUTED BY HASH(product_id) BUCKETS 8
PROPERTIES (
"dynamic_partition.enable" = "true",
"dynamic_partition.time_unit" = "DAY",
"dynamic_partition.start" = "-30",
"dynamic_partition.end" = "7",
"dynamic_partition.prefix" = "p",
"dynamic_partition.buckets" = "8"
);



3. 安装和配置 Kafka
   在 Linux 服务器上执行以下命令安装 Kafka：
   Kafka安装配置脚本

#!/bin/bash

# 下载Kafka
wget https://downloads.apache.org/kafka/3.6.1/kafka_2.13-3.6.1.tgz

# 解压
tar -xzf kafka_2.13-3.6.1.tgz
sudo mv kafka_2.13-3.6.1 /opt/kafka

# 启动Zookeeper
cd /opt/kafka
bin/zookeeper-server-start.sh -daemon config/zookeeper.properties

# 启动Kafka broker
bin/kafka-server-start.sh -daemon config/server.properties

# 等待服务启动
sleep 10

# 创建所需主题
bin/kafka-topics.sh --create --topic postgres-users --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
bin/kafka-topics.sh --create --topic postgres-orders --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
bin/kafka-topics.sh --create --topic sqlserver-products --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
bin/kafka-topics.sh --create --topic sqlserver-customers --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1

# 验证主题创建
bin/kafka-topics.sh --list --bootstrap-server localhost:9092


4. 配置 Flink CDC 同步数据
   4.1 安装 Flink 和必要的连接器
   Flink及CDC连接器安装脚本

#!/bin/bash

# 下载Flink
wget https://archive.apache.org/dist/flink/flink-1.18.0/flink-1.18.0-bin-scala_2.12.tgz

# 解压并移动
tar -xzf flink-1.18.0-bin-scala_2.12.tgz
sudo mv flink-1.18.0 /opt/flink

# 创建连接器目录
mkdir -p /opt/flink/lib

# 下载PostgreSQL CDC连接器
wget https://repo1.maven.org/maven2/com/ververica/flink-connector-postgres-cdc/2.4.1/flink-connector-postgres-cdc-2.4.1.jar -P /opt/flink/lib/

# 下载SQL Server CDC连接器
wget https://repo1.maven.org/maven2/com/ververica/flink-connector-sqlserver-cdc/2.4.1/flink-connector-sqlserver-cdc-2.4.1.jar -P /opt/flink/lib/

# 下载Kafka连接器
wget https://repo1.maven.org/maven2/org/apache/flink/flink-connector-kafka/1.18.0/flink-connector-kafka-1.18.0.jar -P /opt/flink/lib/
wget https://repo1.maven.org/maven2/org/apache/kafka/kafka-clients/3.6.1/kafka-clients-3.6.1.jar -P /opt/flink/lib/

# 启动Flink集群
cd /opt/flink
bin/start-cluster.sh


   4.2 创建 Flink CDC 同步作业
   启动 Flink SQL 客户端并执行同步脚本：
   bash
   /opt/flink/bin/sql-client.sh embedded
   然后在 Flink SQL 客户端中执行以下脚本：
   Flink CDC同步作业SQL

-- 创建PostgreSQL用户表连接
CREATE TABLE postgres_users (
id INT PRIMARY KEY NOT ENFORCED,
name STRING,
email STRING,
created_at TIMESTAMP(3),
age INT
) WITH (
'connector' = 'postgres-cdc',
'hostname' = '你的PostgreSQL主机IP',
'port' = '5432',
'username' = '你的PostgreSQL用户名',
'password' = '你的PostgreSQL密码',
'database-name' = 'cdc_demo',
'schema-name' = 'public',
'table-name' = 'users'
);

-- 创建PostgreSQL订单表连接
CREATE TABLE postgres_orders (
order_id INT PRIMARY KEY NOT ENFORCED,
user_id INT,
product_name STRING,
amount DECIMAL(10,2),
order_date TIMESTAMP(3),
status STRING
) WITH (
'connector' = 'postgres-cdc',
'hostname' = '你的PostgreSQL主机IP',
'port' = '5432',
'username' = '你的PostgreSQL用户名',
'password' = '你的PostgreSQL密码',
'database-name' = 'cdc_demo',
'schema-name' = 'public',
'table-name' = 'orders'
);

-- 创建SQL Server产品表连接
CREATE TABLE sqlserver_products (
product_id INT PRIMARY KEY NOT ENFORCED,
product_name STRING,
price DECIMAL(10,2),
stock INT,
updated_at TIMESTAMP(3)
) WITH (
'connector' = 'sqlserver-cdc',
'hostname' = '你的SQL Server主机IP',
'port' = '1433',
'username' = '你的SQL Server用户名',
'password' = '你的SQL Server密码',
'database-name' = 'cdc_demo',
'schema-name' = 'dbo',
'table-name' = 'products'
);

-- 创建SQL Server客户表连接
CREATE TABLE sqlserver_customers (
customer_id INT PRIMARY KEY NOT ENFORCED,
first_name STRING,
last_name STRING,
phone STRING,
address STRING,
registration_date TIMESTAMP(3)
) WITH (
'connector' = 'sqlserver-cdc',
'hostname' = '你的SQL Server主机IP',
'port' = '1433',
'username' = '你的SQL Server用户名',
'password' = '你的SQL Server密码',
'database-name' = 'cdc_demo',
'schema-name' = 'dbo',
'table-name' = 'customers'
);

-- 创建Kafka输出表 - PostgreSQL用户
CREATE TABLE kafka_postgres_users (
id INT PRIMARY KEY NOT ENFORCED,
name STRING,
email STRING,
created_at TIMESTAMP(3),
age INT
) WITH (
'connector' = 'kafka',
'topic' = 'postgres-users',
'properties.bootstrap.servers' = 'localhost:9092',
'format' = 'json'
);

-- 创建Kafka输出表 - PostgreSQL订单
CREATE TABLE kafka_postgres_orders (
order_id INT PRIMARY KEY NOT ENFORCED,
user_id INT,
product_name STRING,
amount DECIMAL(10,2),
order_date TIMESTAMP(3),
status STRING
) WITH (
'connector' = 'kafka',
'topic' = 'postgres-orders',
'properties.bootstrap.servers' = 'localhost:9092',
'format' = 'json'
);

-- 创建Kafka输出表 - SQL Server产品
CREATE TABLE kafka_sqlserver_products (
product_id INT PRIMARY KEY NOT ENFORCED,
product_name STRING,
price DECIMAL(10,2),
stock INT,
updated_at TIMESTAMP(3)
) WITH (
'connector' = 'kafka',
'topic' = 'sqlserver-products',
'properties.bootstrap.servers' = 'localhost:9092',
'format' = 'json'
);

-- 创建Kafka输出表 - SQL Server客户
CREATE TABLE kafka_sqlserver_customers (
customer_id INT PRIMARY KEY NOT ENFORCED,
first_name STRING,
last_name STRING,
phone STRING,
address STRING,
registration_date TIMESTAMP(3)
) WITH (
'connector' = 'kafka',
'topic' = 'sqlserver-customers',
'properties.bootstrap.servers' = 'localhost:9092',
'format' = 'json'
);

-- 执行同步
INSERT INTO kafka_postgres_users SELECT * FROM postgres_users;
INSERT INTO kafka_postgres_orders SELECT * FROM postgres_orders;
INSERT INTO kafka_sqlserver_products SELECT * FROM sqlserver_products;
INSERT INTO kafka_sqlserver_customers SELECT * FROM sqlserver_customers;




5. 配置 Doris Routine Load
   在 Doris 中创建例行加载任务，从 Kafka 加载数据：
   Doris例行加载配置


-- 连接Doris
-- mysql -h 127.0.0.1 -P 9030 -u root

USE cdc_demo;

-- 创建用户订单数据加载任务
CREATE ROUTINE LOAD cdc_demo.user_orders_load
ON user_orders
PROPERTIES
(
"desired_concurrent_number" = "3",
"max_batch_interval" = "20",
"max_batch_rows" = "300000",
"max_batch_size" = "209715200"
)
FROM KAFKA
(
"kafka_broker_list" = "localhost:9092",
"kafka_topic" = "postgres-users,postgres-orders",
"kafka_partitions" = "0-2,0-2",
"kafka_offsets" = "OFFSET_BEGINNING,OFFSET_BEGINNING"
)
FORMAT AS JSON
(
"json_root" = "$",
"json_paths" = "[\"$.id\",\"$.name\",\"$.order_id\",\"$.product_name\",\"$.amount\",\"$.order_date\"]"
)
COLUMNS (
id, name, order_id, product_name, amount, order_date,
dt = date_format(order_date, '%Y-%m-%d')
);

-- 创建产品客户数据加载任务
CREATE ROUTINE LOAD cdc_demo.product_customers_load
ON product_customers
PROPERTIES
(
"desired_concurrent_number" = "3",
"max_batch_interval" = "20",
"max_batch_rows" = "300000",
"max_batch_size" = "209715200"
)
FROM KAFKA
(
"kafka_broker_list" = "localhost:9092",
"kafka_topic" = "sqlserver-products,sqlserver-customers",
"kafka_partitions" = "0-2,0-2",
"kafka_offsets" = "OFFSET_BEGINNING,OFFSET_BEGINNING"
)
FORMAT AS JSON
(
"json_root" = "$",
"json_paths" = "[\"$.product_id\",\"$.product_name\",\"$.price\",\"$.customer_id\",\"$.first_name\",\"$.last_name\",\"$.registration_date\"]"
)
COLUMNS (
product_id, product_name, price, customer_id, first_name, last_name, registration_date,
customer_name = concat(first_name, ' ', last_name),
dt = date_format(registration_date, '%Y-%m-%d')
);


6. 验证数据同步
   验证 Flink 任务运行状态：访问 Flink Web UI（默认地址：http:// 服务器 IP:8081）查看任务是否正常运行
   验证 Kafka 数据：
   bash
# 查看Kafka主题中的数据
/opt/kafka/bin/kafka-console-consumer.sh --topic postgres-users --bootstrap-server localhost:9092 --from-beginning
验证 Doris 中的数据：
sql
-- 在Doris中查询
SELECT * FROM user_orders LIMIT 10;
SELECT * FROM product_customers LIMIT 10;

-- 检查动态分区是否生效
SHOW PARTITIONS FROM user_orders;
检查 Doris 例行加载状态：
sql
SHOW ALL ROUTINE LOAD;
SHOW ROUTINE LOAD FOR user_orders_load;




