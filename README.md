# go-zero zeromicro
# this project comes from the go-zero.
https://github.com/zeromicro/zero-doc/blob/main/doc/shorturl.md

# require:
## etcd安装：

https://www.jianshu.com/p/964b268e1fc8

## 使用Docker部署单节点etcd

```
export NODE1=0.0.0.0
docker run \
  -p 2379:2379 \
  -p 2380:2380 \
  --volume=/root/docker/etcd/data:/etcd-data \
  --name etcd quay.io/coreos/etcd:latest \
  /usr/local/bin/etcd \
  --data-dir=/etcd-data --name node1 \
  --initial-advertise-peer-urls http://${NODE1}:2380 --listen-peer-urls http://${NODE1}:2380 \
  --advertise-client-urls http://${NODE1}:2379 --listen-client-urls http://${NODE1}:2379 \
  --initial-cluster node1=http://${NODE1}:2380


etcdctl --endpoints=http://localhost:2379 member list
```

## docker安装mysql

```
docker run -itd --name mysql \
-v /data/var/lib/mysql:/var/lib/mysql \
-p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql

```

**docker 安装 mysql 8 版本**

```
# docker 中下载 mysql
docker pull mysql

#启动
docker run --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql

#进入容器
docker exec -it mysql bash

#登录mysql
mysql -u root -p
ALTER USER 'root'@'localhost' IDENTIFIED BY '123456';

#添加远程登录用户
CREATE USER 'u1'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
GRANT ALL PRIVILEGES ON *.* TO 'u1'@'%';
```

## 安装redis

https://hub.docker.com/_/redis/

```
docker run -p 6379:6379 --name zero-redis -v /root/docker/redis/redis.conf:/usr/local/etc/redis/redis.conf  -v /root/docker/redis/data:/data -d redis redis-server /usr/local/etc/redis/redis.conf --appendonly yes


docker run -p 6379:6379 --name zero-redis  -v /root/docker/redis/data:/data -d redis redis-server --appendonly yes

```

# start the shorturl projet.

## vscode root启动
```
code --user-data-dir="~/.vscode-root" --no-sandbox
```

1. 什么是短链服务
短链服务就是将长的 URL 网址，通过程序计算等方式，转换为简短的网址字符串。

写此短链服务是为了从整体上演示 go-zero 构建完整微服务的过程，算法和实现细节尽可能简化了，所以这不是一个高阶的短链服务。

2. 短链微服务架构图
架构图

这里只用了 Transform RPC 一个微服务，并不是说 API Gateway 只能调用一个微服务，只是为了最简演示 API Gateway 如何调用 RPC 微服务而已
在真正项目里要尽可能每个微服务使用自己的数据库，数据边界要清晰
3. goctl 各层代码生成一览
所有绿色背景的功能模块是自动生成的，按需激活，红色模块是需要自己写的，也就是增加下依赖，编写业务特有逻辑，各层示意图分别如下：

API Gateway

api
RPC

架构图
model

model
下面我们来一起完整走一遍快速构建微服务的流程，Let’s Go!🏃‍♂️

4. 准备工作
安装 etcd, mysql, redis

安装 protoc-gen-go

go get -u github.com/golang/protobuf/protoc-gen-go@v1.3.2
安装 protoc

wget https://github.com/protocolbuffers/protobuf/releases/download/v3.14.0/protoc-3.14.0-linux-x86_64.zip
unzip protoc-3.14.0-linux-x86_64.zip
mv bin/protoc /usr/local/bin/
安装 goctl 工具

GO111MODULE=on GOPROXY=https://goproxy.cn/,direct go get -u github.com/tal-tech/go-zero/tools/goctl
创建工作目录 shorturl 和 shorturl/api

mkdir -p shorturl/api

在 shorturl 目录下执行 go mod init shorturl 初始化 go.mod

module shorturl

go 1.15

require (
  github.com/golang/mock v1.4.3
  github.com/golang/protobuf v1.4.2
  github.com/tal-tech/go-zero v1.1.4
  golang.org/x/net v0.0.0-20200707034311-ab3426394381
  google.golang.org/grpc v1.29.1
)
注意：这里可能存在 grpc 版本依赖的问题，可以用以上配置

5. 编写 API Gateway 代码
在 shorturl/api 目录下通过 goctl 生成 api/shorturl.api：

goctl api -o shorturl.api
编辑 api/shorturl.api，为了简洁，去除了文件开头的 info，代码如下：

type (
  expandReq {
    shorten string `form:"shorten"`
  }

  expandResp {
    url string `json:"url"`
  }
)

type (
  shortenReq {
    url string `form:"url"`
  }

  shortenResp {
    shorten string `json:"shorten"`
  }
)

service shorturl-api {
  @server(
    handler: ShortenHandler
  )
  get /shorten(shortenReq) returns(shortenResp)

  @server(
    handler: ExpandHandler
  )
  get /expand(expandReq) returns(expandResp)
}
type 用法和 go 一致，service 用来定义 get/post/head/delete 等 api 请求，解释如下：

service shorturl-api { 这一行定义了 service 名字
@server 部分用来定义 server 端用到的属性
handler 定义了服务端 handler 名字
get /shorten(shortenReq) returns(shortenResp) 定义了 get 方法的路由、请求参数、返回参数等
使用 goctl 生成 API Gateway 代码

goctl api go -api shorturl.api -dir .
生成的文件结构如下：

.
├── api
│   ├── etc
│   │   └── shorturl-api.yaml         // 配置文件
│   ├── internal
│   │   ├── config
│   │   │   └── config.go             // 定义配置
│   │   ├── handler
│   │   │   ├── expandhandler.go      // 实现 expandHandler
│   │   │   ├── routes.go             // 定义路由处理
│   │   │   └── shortenhandler.go     // 实现 shortenHandler
│   │   ├── logic
│   │   │   ├── expandlogic.go        // 实现 ExpandLogic
│   │   │   └── shortenlogic.go       // 实现 ShortenLogic
│   │   ├── svc
│   │   │   └── servicecontext.go     // 定义 ServiceContext
│   │   └── types
│   │       └── types.go              // 定义请求、返回结构体
│   ├── shorturl.api
│   └── shorturl.go                   // main 入口定义
├── go.mod
└── go.sum
启动 API Gateway 服务，默认侦听在 8888 端口

go run shorturl.go -f etc/shorturl-api.yaml
测试 API Gateway 服务

curl -i "http://localhost:8888/shorten?url=http://www.xiaoheiban.cn"
返回如下：

HTTP/1.1 200 OK
Content-Type: application/json
Date: Thu, 27 Aug 2020 14:31:39 GMT
Content-Length: 15

{"shorten":""}
可以看到我们 API Gateway 其实啥也没干，就返回了个空值，接下来我们会在 rpc 服务里实现业务逻辑

可以修改 internal/svc/servicecontext.go 来传递服务依赖（如果需要）

实现逻辑可以修改 internal/logic 下的对应文件

可以通过 goctl 生成各种客户端语言的 api 调用代码

到这里，你已经可以通过 goctl 生成客户端代码给客户端同学并行开发了，支持多种语言，详见文档

6. 编写 transform rpc 服务
在 shorturl 目录下创建 rpc 目录
在 rpc/transform 目录下编写 transform.proto 文件

可以通过命令生成 proto 文件模板

goctl rpc template -o transform.proto
修改后文件内容如下：

syntax = "proto3";

package transform;

message expandReq {
    string shorten = 1;
}

message expandResp {
    string url = 1;
}

message shortenReq {
    string url = 1;
}

message shortenResp {
    string shorten = 1;
}

service transformer {
    rpc expand(expandReq) returns(expandResp);
    rpc shorten(shortenReq) returns(shortenResp);
}
用 goctl 生成 rpc 代码，在 rpc/transform 目录下执行命令

goctl rpc proto -src transform.proto -dir .
注意：不能在 GOPATH 目录下执行以上命令

文件结构如下：

rpc/transform
├── etc
│   └── transform.yaml              // 配置文件
├── internal
│   ├── config
│   │   └── config.go               // 配置定义
│   ├── logic
│   │   ├── expandlogic.go          // expand 业务逻辑在这里实现
│   │   └── shortenlogic.go         // shorten 业务逻辑在这里实现
│   ├── server
│   │   └── transformerserver.go    // 调用入口, 不需要修改
│   └── svc
│       └── servicecontext.go       // 定义 ServiceContext，传递依赖
├── pb
│   └── transform.pb.go
├── transform.go                    // rpc 服务 main 函数
├── transform.proto
└── transformer
    ├── transformer.go              // 提供了外部调用方法，无需修改
    ├── transformer_mock.go         // mock 方法，测试用
    └── types.go                    // request/response 结构体定义
直接可以运行，如下：

$ go run transform.go -f etc/transform.yaml
Starting rpc server at 127.0.0.1:8080...
查看服务是否注册

$ETCDCTL_API=3 etcdctl get transform.rpc --prefix
transform.rpc/7587851893787585061
127.0.0.1:8080
etc/transform.yaml 文件里可以修改侦听端口等配置

7. 修改 API Gateway 代码调用 transform rpc 服务
修改配置文件 shorturl-api.yaml，增加如下内容

Transform:
  Etcd:
    Hosts:
      - localhost:2379
    Key: transform.rpc
通过 etcd 自动去发现可用的 transform 服务

修改 internal/config/config.go 如下，增加 transform 服务依赖

type Config struct {
  rest.RestConf
  Transform zrpc.RpcClientConf     // 手动代码
}
修改 internal/svc/servicecontext.go，如下：

type ServiceContext struct {
  Config    config.Config
  Transformer transformer.Transformer                                          // 手动代码
}

func NewServiceContext(c config.Config) *ServiceContext {
  return &ServiceContext{
    Config:    c,
    Transformer: transformer.NewTransformer(zrpc.MustNewClient(c.Transform)),  // 手动代码
  }
}
通过 ServiceContext 在不同业务逻辑之间传递依赖

修改 internal/logic/expandlogic.go 里的 Expand 方法，如下：

func (l *ExpandLogic) Expand(req types.ExpandReq) (types.ExpandResp, error) {
  // 手动代码开始
    resp, err := l.svcCtx.Transformer.Expand(l.ctx, &transformer.ExpandReq{
      Shorten: req.Shorten,
    })
    if err != nil {
      return types.ExpandResp{}, err
    }

    return types.ExpandResp{
      Url: resp.Url,
    }, nil
    // 手动代码结束
}
通过调用 transformer 的 Expand 方法实现短链恢复到 url

修改 internal/logic/shortenlogic.go，如下：

func (l *ShortenLogic) Shorten(req types.ShortenReq) (types.ShortenResp, error) {
  // 手动代码开始
    resp, err := l.svcCtx.Transformer.Shorten(l.ctx, &transformer.ShortenReq{
  	  Url: req.Url,
    })
    if err != nil {
      return types.ShortenResp{}, err
    }

    return types.ShortenResp{
      Shorten: resp.Shorten,
    }, nil
    // 手动代码结束
}
有的版本生成返回值可能是指针类型，需要自己调整下

通过调用 transformer 的 Shorten 方法实现 url 到短链的变换

至此，API Gateway 修改完成，虽然贴的代码多，但是其中修改的是很少的一部分，为了方便理解上下文，我贴了完整代码，接下来处理 CRUD+cache

8. 定义数据库表结构，并生成 CRUD+cache 代码
shorturl 下创建 rpc/transform/model 目录：mkdir -p rpc/transform/model

在 rpc/transform/model 目录下编写创建 shorturl 表的 sql 文件 shorturl.sql，如下：

CREATE TABLE `shorturl`
(
  `shorten` varchar(255) NOT NULL COMMENT 'shorten key',
  `url` varchar(255) NOT NULL COMMENT 'original url',
  PRIMARY KEY(`shorten`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
创建 DB 和 table

create database gozero;
source shorturl.sql;
在 rpc/transform/model 目录下执行如下命令生成 CRUD+cache 代码，-c 表示使用 redis cache

goctl model mysql ddl -c -src shorturl.sql -dir .
也可以用 datasource 命令代替 ddl 来指定数据库链接直接从 schema 生成

生成后的文件结构如下：

rpc/transform/model
├── shorturl.sql
├── shorturlmodel.go              // CRUD+cache 代码
└── vars.go                       // 定义常量和变量
9. 修改 shorten/expand rpc 代码调用 crud+cache 代码
修改 rpc/transform/etc/transform.yaml，增加如下内容：

DataSource: root:password@tcp(localhost:3306)/gozero
Table: shorturl
Cache:
  - Host: localhost:6379
可以使用多个 redis 作为 cache，支持 redis 单点或者 redis 集群

修改 rpc/transform/internal/config/config.go，如下：

type Config struct {
  zrpc.RpcServerConf
  DataSource string             // 手动代码
  Table      string             // 手动代码
  Cache      cache.CacheConf    // 手动代码
}
增加了 mysql 和 redis cache 配置

修改 rpc/transform/internal/svc/servicecontext.go，如下：

type ServiceContext struct {
  c     config.Config
  Model model.ShorturlModel   // 手动代码
}

func NewServiceContext(c config.Config) *ServiceContext {
  return &ServiceContext{
    c:             c,
    Model: model.NewShorturlModel(sqlx.NewMysql(c.DataSource), c.Cache), // 手动代码
  }
}
修改 rpc/transform/internal/logic/expandlogic.go，如下：

func (l *ExpandLogic) Expand(in *transform.ExpandReq) (*transform.ExpandResp, error) {
  // 手动代码开始
  res, err := l.svcCtx.Model.FindOne(in.Shorten)
  if err != nil {
    return nil, err
  }

  return &transform.ExpandResp{
    Url: res.Url,
  }, nil
  // 手动代码结束
}
修改 rpc/transform/internal/logic/shortenlogic.go，如下：

func (l *ShortenLogic) Shorten(in *transform.ShortenReq) (*transform.ShortenResp, error) {
  // 手动代码开始，生成短链接
  key := hash.Md5Hex([]byte(in.Url))[:6]
  _, err := l.svcCtx.Model.Insert(model.Shorturl{
    Shorten: key,
    Url:     in.Url,
  })
  if err != nil {
    return nil, err
  }

  return &transform.ShortenResp{
    Shorten: key,
  }, nil
  // 手动代码结束
}
至此代码修改完成，凡是手动修改的代码我加了标注

注意：

undefined cache，你需要 import "github.com/tal-tech/go-zero/core/stores/cache"
undefined model, sqlx, hash 等，你需要在文件中
import "shorturl/rpc/transform/model"

import "github.com/tal-tech/go-zero/core/stores/sqlx"
10. 完整调用演示
shorten api 调用

curl -i "http://localhost:8888/shorten?url=http://www.xiaoheiban.cn"
返回如下：

HTTP/1.1 200 OK
Content-Type: application/json
Date: Sat, 29 Aug 2020 10:49:49 GMT
Content-Length: 21

{"shorten":"f35b2a"}
expand api 调用

curl -i "http://localhost:8888/expand?shorten=f35b2a"
返回如下：

HTTP/1.1 200 OK
Content-Type: application/json
Date: Sat, 29 Aug 2020 10:51:53 GMT
Content-Length: 34

{"url":"http://www.xiaoheiban.cn"}
11. Benchmark
因为写入依赖于 mysql 的写入速度，就相当于压 mysql 了，所以压测只测试了 expand 接口，相当于从 mysql 里读取并利用缓存，shorten.lua 里随机从 db 里获取了 100 个热 key 来生成压测请求

Benchmark

可以看出在我的 MacBook Pro 上能达到 3 万 + 的 qps。

12. 完整代码
https://github.com/zeromicro/zero-examples/tree/main/shorturl

12. 总结
我们一直强调 工具大于约定和文档。

go-zero 不只是一个框架，更是一个建立在框架 + 工具基础上的，简化和规范了整个微服务构建的技术体系。

我们在保持简单的同时也尽可能把微服务治理的复杂度封装到了框架内部，极大的降低了开发人员的心智负担，使得业务开发得以快速推进。

通过 go-zero+goctl 生成的代码，包含了微服务治理的各种组件，包括：并发控制、自适应熔断、自适应降载、自动缓存控制等，可以轻松部署以承载巨大访问量。