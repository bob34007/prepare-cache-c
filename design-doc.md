# 一个缓存 MySQL C API prepare stmt 的 demo 库 #

## 需求描述 ##

1. 用户类C程序调用C API访问 MySQL 数据库；
2. 调用流程 prepare -> execute -> close ;
3. MySQL / TiDB 不会像 Oracle OCI 那样缓存 stmt；
4. 下次再 prepare 仍然会走流程，造成性能较差，甚至比普通 query 还差；若不调用 close ，由于分层设计，容易造成内存泄漏。
5. 希望实现缓存库，对外暴漏类似 MySQL 接口，用户调用 close 时，不真正关闭；而是缓存到内存中；后续 prepare 再来时直接返回 stmt，缓存通过时间淘汰，越早缓存的越先淘汰；


## 解决方案 ## 

1. 缓存内存结构设计

#define cache_size 2000 

int64 g_conn_id = 0 ;
pthread_mutex_t g_conn_id_mutex;

MYSQL->unused1 存储connid



typedef struct{
	int key;        //键
	void *val ;  //值
}data_type; //对基本数据类型进行封装，类似泛型

typedef struct{
	data_type data;
	struct hash_node *next;  //key冲突时，通过next指针进行连接
}hash_node;

typedef struct{
	int size;
	pthread_mutex_t mutex;//get和set、size时锁保护，防止并发写错误
	hash_node *table;
}hash_map;

typedef struct{
	MYSQL_STMT *stmt;
	int64 stmt_id;
	pthread_mutex_t mutex;
	bool is_used; // true 使用中，false 未使用；节点删除和保活时使用
	char *prepare_sql;
	int prepare_sql_len ;
}val_stmt;


typedef struct{
	MYSQL *conptr;
	int64 last_ping_time ;
	int64 last_use_time ;
	int64 connid ;
	pthread_mutex_t mutex;
	bool is_used; // true 使用中，false 未使用；节点删除和保活时使用
}val_mysql;


//提供功能 
缓存 sql prepare 句柄，通过 stmtid+sql 生成 hash key 作为缓存 key 

普通 sql 通过 mysql client 执行 ；


客户端链路保活 
1. 定时（10 ms）遍历所有客户端缓存结构中的客户端执行 mysql_ping 命令

按照 客户端 进行所有 stmt 关闭
提供 stmt 关闭 和缓存 清理 功能




                               