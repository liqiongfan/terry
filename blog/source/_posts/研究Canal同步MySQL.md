---
title: 研究Canal同步MySQL
date: 2020-08-02 20:22:27
tags:
- canal
- binlog
- mysql
- 同步中间件
categories:
- tech
description: canal同步中间件
---

### 什么是Canal？
Canal是基于 MySQL 数据库增量日志解析，提供增量数据订阅和消费，我们可以用canal作为同步中间件，进行数据到数据仓库(doris|hive)等的落地

### 工作原理
Canal是基于mysql的binlog二进制日志来进行的数据处理，可以做到数据的绝对落地，只要保证高可用，数据肯定是最终一致性
Canal 模拟 MySQL slave 的交互协议，伪装自己为 MySQL slave ，向 MySQL master 发送dump 协议
MySQL master 收到 dump 请求，开始推送 binary log 给 slave (即 canal )
canal 解析 binary log 对象(原始为 byte 流)

### 部署示例

+ 环境准备

```shell script
[mysqld]
log-bin=mysql-bin # 开启 binlog
binlog-format=ROW # 选择 ROW 模式
server_id=1       # 配置 MySQL replication 需要定义，不要和 canal 的 slaveId 重复
```

+ 设置授权账号，用作 `Canal` 的同步账号：

```shell script
CREATE USER canal IDENTIFIED BY 'canal';  
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
--- GRANT ALL PRIVILEGES ON *.* TO 'canal'@'%' ;
FLUSH PRIVILEGES;
```

+ 下载 `Canal`，或者通过如下的地址下载

```html
https://github.com/alibaba/canal/releases/download/canal-1.1.4/canal.deployer-1.1.4.tar.gz
```

+ 解压和更新配置

```shell script
vi conf/example/instance.properties
```

```shell script
## mysql serverId
canal.instance.mysql.slaveId = 1234             # 这个id必须不同于 mysql server_id
#position info，需要改成自己的数据库信息
canal.instance.master.address = 127.0.0.1:3306  # 需要同步的mysql的ip和端口
canal.instance.master.journal.name =            # binlog 文件名 通过 show master status 得到
canal.instance.master.position =                # binlog 位置  通过 show master status 得到
#username/password，需要改成自己的数据库信息
canal.instance.dbUsername = canal               # mysql授权的同步账号
canal.instance.dbPassword = canal               # mysql授权的同步密码
canal.instance.defaultDatabaseName =            # 登陆的数据库名
canal.instance.connectionCharset = UTF-8
#table regex
canal.instance.filter.regex = .\*\\\\..\*       # 所有的表
```

+ 启动canal

```shell script
sh bin/startup.sh
```

重启：

```shell script
sh bin/restart.sh
```

+ 错误日志

如果存在错误，那么可以查看日志： `logs/example/example.log`

```shell script
2020-08-02 20:06:27.854 [main] WARN  c.a.o.canal.parse.inbound.mysql.dbsync.LogEventConvert - --> init table filter : ^.*\..*$
2020-08-02 20:06:27.855 [main] INFO  c.a.otter.canal.instance.core.AbstractCanalInstance - start successful....
2020-08-02 20:06:27.870 [destination = example , address = /19.118.184.34:33006 , EventParser] WARN  c.a.o.c.p.inbound.mysql.rds.RdsBinlogEventParserProxy - ---> begin to find start position, it will be long time for reset or first position
2020-08-02 20:06:27.989 [destination = example , address = /19.118.184.34:33006 , EventParser] WARN  c.a.o.c.p.inbound.mysql.rds.RdsBinlogEventParserProxy - prepare to find start position mysql-bin.000003:291:1596366589000
2020-08-02 20:06:28.024 [destination = example , address = /19.118.184.34:33006 , EventParser] WARN  c.a.o.c.p.inbound.mysql.rds.RdsBinlogEventParserProxy - ---> find start position successfully, EntryPosition[included=false,journalName=mysql-bin.000003,position=291,serverId=1,gtid=,timestamp=1596366589000] cost : 145ms , the next step is binlog dump
```
看到上面的输出就证明成功了，如果有错误，那么对症下药即可：

+ 客户端的同步代码，这边以 `go` 作为客户端语言作为演示

```go go
package main

import (
	"fmt"
	"log"
	"os"
	"time"

	"github.com/golang/protobuf/proto"
	"github.com/withlin/canal-go/client"
	protocol "github.com/withlin/canal-go/protocol"
)

func main() {

	// 192.168.199.17 替换成你的canal server的地址
	// example 替换成-e canal.destinations=example 你自己定义的名字
	connector := client.NewSimpleCanalConnector(
		"127.0.0.1", 11111,
		"", "",
		"example", 60000, 60*60*1000 )
	err := connector.Connect()
	if err != nil {
		log.Println(err)
		os.Exit(1)
	}

	// https://github.com/alibaba/canal/wiki/AdminGuide
	//mysql 数据解析关注的表，Perl正则表达式.
	//
	//多个正则之间以逗号(,)分隔，转义符需要双斜杠(\\)
	//
	//常见例子：
	//
	//  1.  所有表：.*   or  .*\\..*
	//	2.  canal schema下所有表： canal\\..*
	//	3.  canal下的以canal打头的表：canal\\.canal.*
	//	4.  canal schema下的一张表：canal\\.test1
	//  5.  多个规则组合使用：canal\\..*,mysql.test1,mysql.test2 (逗号分隔)

	err = connector.Subscribe(".*\\..*")
	if err != nil {
		log.Println(err)
		os.Exit(1)
	}

	for {

		message, err := connector.Get(100, nil, nil)
		if err != nil {
			log.Println(err)
			os.Exit(1)
		}
		batchId := message.Id
		if batchId == -1 || len(message.Entries) <= 0 {
			time.Sleep(300 * time.Millisecond)
			// fmt.Println("===没有数据了===")
			continue
		}

		printEntry(message.Entries)

	}
}

func printEntry(entrys []protocol.Entry) {

	for _, entry := range entrys {
		if entry.GetEntryType() == protocol.EntryType_TRANSACTIONBEGIN || entry.GetEntryType() == protocol.EntryType_TRANSACTIONEND {
			continue
		}
		rowChange := new(protocol.RowChange)

		err := proto.Unmarshal(entry.GetStoreValue(), rowChange)
		checkError(err)
		if rowChange != nil {
			eventType := rowChange.GetEventType()
			header := entry.GetHeader()
			fmt.Println(fmt.Sprintf("================> binlog[%s : %d],name[%s,%s], eventType: %s", header.GetLogfileName(), header.GetLogfileOffset(), header.GetSchemaName(), header.GetTableName(), header.GetEventType()))

			for _, rowData := range rowChange.GetRowDatas() {
				if eventType == protocol.EventType_DELETE {
					printColumn(rowData.GetBeforeColumns())
				} else if eventType == protocol.EventType_INSERT {
					printColumn(rowData.GetAfterColumns())
				} else {
					fmt.Println("-------> before")
					printColumn(rowData.GetBeforeColumns())
					fmt.Println("-------> after")
					printColumn(rowData.GetAfterColumns())
				}
			}
		}
	}
}

func printColumn(columns []*protocol.Column) {
	for _, col := range columns {
		fmt.Println(fmt.Sprintf("%s : %s  update= %t", col.GetName(), col.GetValue(), col.GetUpdated()))
	}
}

func checkError(err error) {
	if err != nil {
		fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())
		os.Exit(1)
	}
}
```

输出如下：

```shell script
================> binlog[mysql-bin.000003 : 5006],name[blog,cs_comments], eventType: UPDATE
-------> before
coid : 5  update= false
cid : 89  update= false
created : 1566226589  update= false
author : 不拍片  update= false
authorId : 1  update= false
ownerId : 1  update= false
mail : xeapplee@gmail.com  update= false
url : https://www.supjos.cn  update= false
ip : 14.28.4.208  update= false
agent : Mozilla/5.0 (iPhone; CPU iPhone OS 12_4 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/12.1.2 Mobile/15E148 Safari/604.1  update= false
text : 只是想丰富一下PHP的原生功能而已，自己动手丰衣足食  update= false
type : comment  update= false
status : approved  update= false
parent : 4  update= false
gravatar :   update= false
-------> after
coid : 5  update= false
cid : 89  update= false
created : 1566226589  update= false
author : 不拍片  update= false
authorId : 0  update= true
ownerId : 1  update= false
mail : xeapplee@gmail.com  update= false
url : https://www.supjos.cn  update= false
ip : 14.28.4.208  update= false
agent : Mozilla/5.0 (iPhone; CPU iPhone OS 12_4 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/12.1.2 Mobile/15E148 Safari/604.1  update= false
text : 只是想丰富一下PHP的原生功能而已，自己动手丰衣足食  update= false
type : comment  update= false
status : approved  update= false
parent : 4  update= false
gravatar :   update= false
================> binlog[mysql-bin.000003 : 5826],name[test_user,], eventType: QUERY
================> binlog[mysql-bin.000003 : 6062],name[test_user,user_test], eventType: CREATE
================> binlog[mysql-bin.000003 : 6540],name[test_user,user_test], eventType: INSERT
id : 1  update= true
name : hello  update= true
```

可以看到数据同步完整成功了
