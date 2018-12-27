[base业务框架](https://github.com/microsvs/base)使用手册

# 开发手册

本项目是对当前使用base业务框架的开发人员，提供一份完整的开发手册，方便开发人员使用。

> 希望本文档能够给大家带来方便。同时业务框架在不断更新中...， 如果文档没有及时更新，也可以email或者提相关的issue

## 编写微服务

采用[demo](https://github.com/microsvs/demo)项目，作为初始项目，然后修改为自己的项目。

总体推荐一个新创建的产品，项目结构目录推荐以下呈现方式：

```shell
chen:demo chenchao$ ls
README.md       common          glide.lock      token           vendor
address         gateway         glide.yaml      user
chen:demo chenchao$ ls gateway/
gateway         main.go         r_address.go    r_errors.go     r_me.go         r_token.go      schema.go
chen:demo chenchao$ ls address/
controllers     main.go         models          schema.go
chen:demo chenchao$ ls common/
consts  rpc     types   utils
```

1. 对于项目目录来说，比如：demo是由四个微服务加上一个`common`目录构成，其他是相关依赖包。一个项目必然有一个网关服务，对外暴露API的；
2. 对于gateway网关服务，它是直接转发web的请求访问，同时可能会做一些身份登录校验，校验token的有效性；以及可以做鉴权和流控；不过后者可以在上istio后，流控就可以业务不关心了
3. 其他微服务代码结构，则是由`main.go, schema.go, types, controllers, models`五个文件或者目录构成；
4. `common`目录，主要用于一些schema的资源定义，公共组件，错误码注册，微服务端口和名称定义、rpc微服务调用和常量定义。

> 对于上面的第3点需要说明的是，types目录可能也不需要，它主要定义schema的资源对象，如果gateway需要与微服务共用，就需要写入到common目录中

> 这里说明一点, 理论上微服务容器化后，所有微服务端口都可以是相同的，但是我们为了兼容非容器化，所以微服务端口是枚举型的，从8081开始进行iota。

## 环境变量

| 名称 | 值 | 描述 |
|---|---|---|
| APP_ZK | 服务地址| 配置中心地址，默认值：127.0.0.1:2181 |
| APP_NAME | 项目名称 | 产品名称 |
| APP_LOG | log绝对路径 | 默认：/var/log/app |
| APP_ENV | 部署环境 | 本地环境、开发、测试和生产, 默认：developer |
| APP_VERSION | 产品版本 | 默认: v1.0|
| APP\_TRACER\_AGENT | jaeger agent地址 | 默认值: 0.0.0.0:6831 |


## graphql常用方法

graphql是facebook发明的，对外提供API的两种方式：RESTful与graphql，各有优劣；graphql最大的优势所见即所得，协议即接口。

下面给出的常用方法通常是rpc调用后，需要进行graphql变量值与golang变量值的相互转化。

```shell
// FixTypeFromGoToGraphql方法
// @params0: rpc返回的数据
// @params1: rpc返回的数据graphql schema资源对象定义
// 含义：把graphql资源对象数据转化为golang变量值
// 比如：graphql中的枚举Enum定义，需要转化为golang中的枚举定义
// 比如：graphql中的资源列表定义，需要转化为golang中的列表定义，因为在graphql中的列表都是interface类型的，需要转化为golang中的资源定义的已知类型
// ...
**func FixTypeFromGoToGraphql(v interface{}, argType graphql.Input) (result interface{})**

// FixTypeFromGraphqlToGo方法, 与上面的方法作用相反
// @param0: 
func FixTypeFromGraphqlToGo(data interface{}, t graphql.Output) interface{}
```

这里提供了几个有关这两个方法的单元测试：

```shell
 cd $GOPATH/src/github.com/microsvs/base
 
 go test -cover=true -run TestFixTypeFromGraphqlToGoSimple
 
 # 上面单元测试返回结果：
10，20，30
 
go test -cover=true -run TestFixTypeFromGraphqlToGoComplex 

# 上面单元测返回结果：
map[created_at:2019-11-24 03:29:08 +0800 CST name:kim age:16 vehicle:10 workers:[map[company:alibaba position:30] map[company:tencent position:40]] updated_at:2019-11-24 03:29:08 +0800 CST]
```

通过上面两个单元测试，我们可以明白FixTypeFromGraphqlToGo是用来把graphql数据转化为golang识别的数据类型；

**注意，在rpc调用过程中，我们使用的还是graphql数据模型的通信，所以需要在请求数据和响应数据前后进行相关类型的数据转换，比如：枚举类型、时间类型和其他不相同形式的数据转换**

```shell
 cd $GOPATH/src/github.com/microsvs/base

go test -cover=true -run TestFixTypeFromGoToGraphqlSimple

# 上面单元测试返回结果
CEO

go test -cover=true -run TestFixTypeFromGoToGraphqlComplex

# 上面单元测试返回结果：
map[created_at:2018-12-27T14:12:49.523371245+08:00 updated_at:2018-12-27T14:12:49.523371228+08:00 name:"kim" age:16 vehicle:Bike workers:[map[company:"alibaba" position:CEO] map[company:"tencent" position:CTO]]]
```

通过上面两个单元测试，我们可以明白FixTypeFromGoToGraphql方法是用来把go类型数据转化为graphql数据；

注意graphql类型数据与golang类型数据的相互转化后的数据对比，看看哪里数据有变化

------------------

// 方法用于隐藏不想展示给调用方的字段数据，比如，有些密码、token等隐私数据。
> func HideGLFields(obj *graphql.Object, v ...string) *graphql.Object


// 方法用于gateway直接调用后端，复用请求，指定目标服务
> func RedirectRequest(p graphql.ResolveParams, target rpc.FGService) (interface{}, error)

// 修改gateway接收的请求，然后转发给指定目标服务
> func RedirectRequestEx(
    p graphql.ResolveParams,
    exArgs map[string]interface{},
    excommon map[string]graphql.Input,
    targetService rpc.FGService,
    targetObj interface{}) (interface{}, error)

该方法使用场景在于，当我们需要组织后端服务收集到的数据，然后再转发到其他微服务时，就需要构建一个请求

> `base.RedirectRequestEx(
        "QUERY/MUATION",
        p,
        map[string]interface{}{"user_id": user.ID},
        map[string]graphql.Input{"user_id": graphql.String},
        base.FGSUser,
        "beat_map_type",
        nil)`

// GLObjectFields获取graphql.Object的所有字段, 作为graphql Query/Mutation的返回值
> func GLObjectFields(obj *graphql.Object) string


## 配置中心

配置中心，主要是对项目需要用到的初始化或者环境配置，都可以放到配置中心；包括但不限于：项目名、版本号、服务部署环境、cache配置、db配置、mq配置、dns配置和其他参数配置

每个微服务都单独配置各项所用到的配置，该业务框架认为各个微服务配置都是不一样的。

对于存储配置，包括cache和db，都是主从配置master和slave，一主多从的配置方式；可以不填写slave配置, 则slave自动共用master配置

slave可以指定多个，配置由`,`逗号分隔slave。

目前环境氛围四个环境：developer本地开发环境，dev-开发环境、test-QA环境、prod-生产环境

### 缓存配置

// 缓存，比如cache，可以指定密码和没有密码两种方式， 当不指定redis密码时，则`@`非必空
> /demo/v1.0/dev/cache/user=master=redis://auth-pass@127.0.0.1:6379

> /demo/v1.0/dev/cache/user=master=redis://auth-pass@192.168.1.100:6379,redis://auth-pass@192.168.1.101:6379,192.168.1.102:6379

### db

// db存储，比如mysql
> /demo/v1.0/dev/db/user=master=username:password@tcp(192.168.1.101:3306)/demo?charset=utf8&parseTime=True&loc=Local

// mongodb
> /demo/v1.0/dev/mongo/user=master=username:password@192.168.1.101:3000/demo

### 消息队列

> /demo/v1.0/dev/mq/user=maste=amqp://username:password@192.168.1.101:3001/queue?heartbeat=15

### dns

> /demo/v1.0/dev/dns=gateway.api.xhj.com=192.168.1.101:8081

注意dns的key为xxx.api.xhj.com, `xhj`是表示base业务框架作者的江湖称呼：小黄鸡；value可以为service的高可用配置, 比如: VIP

### 参数配置

项目除了通用配置外，可能还会遇到其他参数配置，比如设置feature开关等, oss路径，sms配置等

### 配置存储

1. 当微服务第一次从配置中心获取，需要一次冷加载，可以设置在微服务启动时，也就是程序的init方法中，初始化将要使用的存储或者其他配置；
2. 该业务框架使用了本地缓存，存储配置，当微服务再次获取配置时，则直接在本地缓存中获取的。
3. 同时第一次从配置中心获取配置后，就开始了监听配置中心的key，并更新到本地缓存中。如果是存储相关包括db、cache、mq、config等会监听到配置变化后，会自动重连各个存储server。则对业务是无感知的

## 日志模块

base业务框架的日志模块，是采用的本地落日志。这里讲下我做过的日志优化相关工作。

```shell
现象： 因为我在使用该base业务框架进行业务上线后，我们使用的k8s集群，微服务经常由于内存占用过大（100Mi），导致被杀，然后又涨又杀。我发现微服务本身只占用了20M的内存。

原因：k8s监控pod，不仅仅是微服务进程本身的内存占用；它是对container内存资源占用的总和，我监控container，top发现cache/buffer不断升高，后来查了文档，发现k8s是统计的cgroup内存资源占用，所以导致的container被杀后，k8s又重新调度。

解决思路：在本地落日志时，操作系统认为落日志的文件会立即读取，所以操作系统会在日志落到磁盘之前，会存储到page cache中，这样page/cache数值越来越大。

解决方法：在落日志时，我们业务框架使用bytes.buffer进行4kb的缓存，每次写入4kb，也就是一页的数据量，这种写入的方式是直接IO方式，这样就不会在page cache进行日志缓存了。os.DIRECT_IO，又因为各个平台底层架构不同，对于macos用的flag与linux用的flag不同等原因，最后就是用的一个简单封装库directio。

效果：优化后，效果非常理想，现在线上业务一直在跑，但是内存占用基本上稳定不变。

后话：cache/buffer是可以重复利用的内存，只是对于k8s集群平台，它是监控的cgroup资源，所以遇到了这个问题。
```

### 日志使用方法

日志的使用，不需要初始化，调用即初始化。

提供的方法有: `log.ErrorRaw`, `log.Debug`, `log.Info`等方法可以使用。

日志写入到的文件路径：环境变量${APP_LOG}，默认/var/log/app目录下，再根据微服务名称进行日志分文件夹。日志目录层次可以自己调整，目前业务默认支持的日志路径:

**/var/log/app/${service name}/YYYYMM/DD/${service name}_YYYYMMDD.log**


> 备注：当日志落地到本地后，你可以通过启动一个agent日志收集服务，比如fluentd，ELK。

## db模块

db模块，我们使用了一个DAL，数据访问层，屏蔽了底层db存储的细节，底层支持PostgreSQL, MySQL, SQLite, MSSQL, QL and MongoDB;

大家可以查看这个DAL数据访问层的github库，[upper/db](https://github.com/upper/db), 自认为比其他的orm好用很多。

对于数据库的初始化，我们可以微服务启动时，在init方法中进行存储client的初始化连接
> db.InitDB(rpc.FGSUser)

然后在业务需要使用的时候，比如：更新、新增和删除操作

// 从slave获取一个db连接
>   db.SlaveDB(rpc.FGSUser)
// 从master获取一个db连接
>   db.MasterDB(rpc.FGSUser)

## 缓存模块

对于redis缓存，不需要业务端做相关的连接释放动作。并提供了对缓存的interface。

```shell
type Connection interface {
    Set(key string, value interface{}) error
    Get(key string) (interface{}, error)
    Del(key string) error
    Expire(key string, sec int) error
    Exist(key string) (bool, error)
    TTL(key string) (time.Duration, error)
    Close() error
    ComplexCmd(cmd string, values ...interface{}) (interface{}, error)
}
// 这个interface前6个方法基本上满足了90%的需求，最后一个方法ComplexCmd则是对复杂需求的缓存操作命令。
```

初始化和获取缓存连接的方法，同db操作

## 消息队列模块

提供了两个初始化和监听消费队列消息的API。

// 初始化消息队列连接
> func InitMQ(service rpc.FGService)

// 使用默认的方法监听消费队列产生的消息
> func ConsumeDefault(service rpc.FGService, queue string) <-chan amqp.Delivery

// 同时提供了一个多配置监听消费队列产生的消息
> func Consume(service rpc.FGService, queue, consumer string, autoAck, exclusive, noLocal, noWait bool, args amqp.Table) <-chan amqp.Delivery

> 消息队列client具有重连机制

## 错误码

base业务框架本身已经提供了一些错误码，我们可以通过gateway的errors查询接口，去查看这些错误码及其含义。

比如：系统错误，客户端请求错误。可以在`$GOPATH/src/github.com/microsvs/base/pkg/errors/const.go`文件中查看。

同时如果业务微服务需要其他错误码，则需要进行错误码的注册, 通过调用`pkg/errors/register.go`文件中的Register方法进行错误码注册。

// 业务微服务错误码注册
> func Register(errMap map[FGErrorCode]string) error

## 服务注册

base业务框架本身也提供了一些服务，包括：网关、签名、流控、用户、token、地址和图片共7个服务。如果还需要其他微服务，则通过RegisterService方法注册。

文件路径：`$GOPATH/src/github.com/microsvs/base/pkg/rpc/service.go`

> func RegisterService(service FGService, serviceName string) error

## rpc调用

当微服务之间需要进行rpc调用时，框架提供了一个API进行跨服务调用。

// dns参数值可以通过base.Service2Url(service)获取
> func CallService(ctx context.Context, dns string, data string) (map[string]interface{}, error)


## 常用方法

在`$GOPATH/src/github.com/microsvs/demo/common/utils`目录下提供了一些常用方法。比如：

1. http调用, HttpPostBody和HttpPostJson API；
2. 提供了阿里云oss存储封装好的API；
3. 对于所有graphql请求的参数必填校验API，CheckAndAssignParams，该方法的实用性非常高
4. ArrayToString方法，slice类型的数据，按照指定格式进行字符串化;
5. 生成4位的短信验证码，GenerateVerifyCode
6. GenerateUUID方法，生成唯一的uuid
7. 还提供了一个获取时间变量timer.Now，由一个goroutine进行托管, 这样系统不用频繁去调用时间

```shell
// 必填参数校验，并把数据返回
if err = utils.CheckAndAssignParams(p.Args, map[string]interface{}{
    "sale_order_id": &saleOrderId,
    "vehicle_no":    &vehicleNo,
}); err != nil {
    return false, err
}
```

## 身份携带

最后一个需要说明的小点：

> 有身份的用户在跨微服务调用时，都会把自身的身份带到context中，这样直接从context中获取token、mobile和其他相关信息是非常方便的。

使用方式如下所示：

```shell
if _, ok = p.Context.Value(rpc.KeyUser).(*itypes.User); !ok {
    log.ErrorRaw("[GetAdOnlineLaunchStatis] get user from context failed.")
    return nil, errors.FGEInvalidToken
}
```

通过上面的这种方式，我们就可以校验用户是否有登录态，context目前支持的key有

```shell
    KeyRPCID
    KeyService
    KeyRawRequest
    KeyMobile
    KeyUser
    KeyConsoleInfo
    KeyProtocalType
```
