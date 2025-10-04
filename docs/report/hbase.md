# . CDH 安装 hbase
# 创建 hbase 表, 数据由 Flink 写⼊ FlinkAPI 或者 FlinkSQL 都可以
# hbase rowkey 的设计, 符合企业原则
# hbase 数据进⼊后, 使⽤ hive 和 hbase 的映射表进⾏映射, 数据需要可查询, 实现可分区为佳

1.进入 HBase Shell 交互界面：
hbase shell

2.创建命名空间
命名空间用于隔离表（类似数据库的库），避免表名冲突：
shell
# 创建命名空间
create_namespace 'user_behavior'

# 查看所有命名空间
list_namespace

示例 1：简单创建表创建一个名为user_behavior:track_log的表，包含 2 个列族info和action：

create 'user_behavior:track_log', 'info', 'action'
示例 2：带属性的列族配置企业级场景中，通常会配置列族的属性（如过期时间、压缩算法等）：

create 'user_behavior:track_log',
{NAME => 'info', TTL => '2592000', COMPRESSION => 'SNAPPY', VERSIONS => 3},  # 列族1：info
{NAME => 'action', TTL => '604800', BLOCKSIZE => 65536}                       # 列族2：action


#常用列族属性说明：
TTL：数据过期时间（秒），过期后自动删除（如2592000=30 天）
COMPRESSION：压缩算法（如SNAPPY、GZ，节省存储空间）
VERSIONS：保留的版本数（默认 1，需多版本查询时可增大）
BLOCKSIZE：数据块大小（默认 64KB，大字段可调大）
IN_MEMORY：是否优先加载到内存（true/false，热点列族可设为true）


3. 验证表是否创建成功
# 查看表是否存在
exists 'user_behavior:track_log'
# 查看表结构（列族及属性）
describe 'user_behavior:track_log'
# 列出所有表
list


4.高级操作：预分区表
默认情况下，HBase 表只有 1 个 Region，数据量大时会自动分裂，可能导致热点问题。企业级场景通常会预分区，提前划分 Region：
# 语法：create '表名', '列族', {SPLITS => ['分裂点1', '分裂点2', ...]}
# 示例：按RowKey前缀预分区（假设RowKey以哈希前缀开头）
create 'user_behavior:track_log', 'info', {
SPLITS => ['00000000', '20000000', '40000000', '60000000', '80000000']
}


5.删除表
若需删除表，需先禁用（disable）再删除（drop）：
disable 'user_behavior:track_log'  # 禁用表
drop 'user_behavior:track_log'     # 删除表




