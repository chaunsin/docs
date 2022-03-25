# restful的个人的一些看法讨论

# 引言

什么是良好restful的设计,我个人的看法是:见名知意通俗易懂,在实际开发中便捷方便,理论经得起考验的.本文以开发者的角度去讨论restful相关问题。
另外一些相关设计规范文章什么的,我就不在这里面详细的去讲解了,网上有很多文章可参考,也可以查看本文底部的先关参考链接。本文只说下我所使用中我所看的一些 相对来说不太合理的点,那些良好的规范没有争议的我就不在这里列举了

# 问题

> 一、url中不能使用版本号的

看了大部分博文总结出方案如下

```
1. 请求头中 Accept:application/json+v1
2. 自定义请求头 X-Api-Version:1
3. GET /v1/users/1
```

这三种方案通俗易懂,都能控制api版本,但我认为最优的方式是3,为什么？先说下实际使用场景问题

方案一和方案二:

比如查询用户信息接口,现在因为需求变更,我们增加新的版本接口v2去实现,我们在请求头加了版本信息控制，ok那么问题来了,按照restful的设计理念这个接口需要保留旧的代码, 然后在加入新的代码,这会有两个问题

1. 代码可能会存在耦合的问题,controller层代码会非常的臃肿,这就会引发出另一个问题,后端代码设计(mvc)的问题,c层几乎只做简单的参数逻辑校验,但现在加入了逻辑处理。
2. 业务逻辑变的复杂.比如不使用请求头的逻辑是:http收到请求后解析参数做参数校验,然后在调用逻辑层的代码进行后续处理,现在加入请求头你需要怎么处理？逻辑会变成如下:
    1. 接受http请求然后判断请求头的参数
    2. 执行case条件v1执行v1代码,v2执行v2方法
    3. 执行不同版本的参数校验
    4. 调用逻辑层代码以及后续逻辑处理

方案三:

方案三更适合，为什么更适合？

1. 没有校验请求头这一步骤跟以上相比
2. 通过路由控制资源,v1/user和v2/user相当于两个不同的资源来进行对待,代码相对解耦方法隔离
3. url可读性更高.在和前端联调以及开发者自己测试便捷许多,见名之意,直接找到对应版本文档即可,不用考虑这个user接口请求头中传入v1和v2版本差异应该传什么值多值漏值值冲突等问题,
   以及在撰写文档时更加便捷可分开处理对待.可读性是很重要的在实际工作中能省去很多沟通成本.
4. 沟通成本与学习成本(ps:我不是反对学习).url中加入版本方式是市面上比较流行的一种方式,差不多得到了共识。这很重要,每个人的水平能力以及所处的环境工作经历等原因不同,比如其他公司跟我们所在的公司进行合作对接调用
   接口,可能你公司使用的是上文中方案1和方案2的方式,相对较冷门的方式,或者说对方公司没有用过这种方式,以及初学者等等吧他可能不能立马理解你设计的这种方式,
   这这就增加了讲解以及沟通成本,如果你有幸做过甲方的项目,当他们不接受这种方式时,需要改需求或者重新实现一套代码,你一定能理解这种痛苦.

> 2. 过滤条件

说下常见的多条件列表的查询条件

```
// 一个等于30岁的列表查询且有分页以及降序排列的方式返回数据
GET user?age=30&limit=10&offset=1&sort=desc
```

ok上述这个查询没问题很清楚了,简单明了.现在我们有样的复杂查询需求大概查询条件是这样的(摘自阮一峰博文中的一个兴杰用户的评论[1])如下

```
>= , between , || , (color=red && size=s)
```

ok那么按照restful的规定方式,url该怎么设计呢,我们还是以上述例子中&字符的方式来进行参数传递,这没有问题,但是如果查询条件中有数组列表的查询需求你该怎么处理呢？
最常见的方式可能就是id中间以逗号或其他分割符的形式来区分每个id,这种方式当id过多(比如查询20个id,id为uuid形式),会造成url过长,过长会有以下问题:

1. 部分浏览器会限制url的长度大小,造成参数截断,当然实际业务中几乎达不到浏览器限制的大小条件
2. 可读性极差,在日常前端后端以及测试人员拷贝粘贴以及参数修改模拟测试等场景都是非常恶心的问题。你想想一个超长的url给到你,你把它粘贴到浏览器中，
   你屏幕大小是有限的你仅仅想更改其中一个参数值,此时你只能通过键盘前进和后退键移动到指定参数位置然后在进行修改,或者一直拖动着鼠标,就问你累不？

为了缩短url长度,可能这时候你会有另一种想法直接写条件表达式可以么这种复杂场景？(restful没有针对这种情况的说明)姑且说可以但是你服务端你解析业务逻辑该怎么写,
前端该怎么根据用户的选中的条件生成条件表达式,以及当发生了重大的业务升级参数大变更代码测试维护都是一个问题。

**那参数过长或者复杂条件的查询就没有相对很好的方案了么？**

有的.

其实我们可以这样做用post+json的方式.这一点是违背了restful设计的原则的.先说下变动

1. restful规定GET必须是查询数据操做,现在需要换成POST形式
2. url中的查询参数条件放到了body中,使用json或者其他结构化的协议方式存储表达查询参数条件

那么这么做的好处是什么？以及观点

1. 复杂的查询条件,可任意书编写,使用json这种方式可读性表达性更强.
2. 放入到body中,POST没有url参数长度限制这一说,另外有数组多个id查询条件时没有任何约束限制

现在我们破坏了restful的规则.那么按照restful规则设计的接口,可能会与之前的接口会有冲突问题比如我们原有两个接口

```
// 一个多条件复杂的用户列表查询
GET /user?ids=111,222,333&age=30&limit=10&offset=1&sort=desc&.&.&.&.......

// 一个单用户创建接口
POST /user
```

现在变更为

```
// 列表查询
POST /user 
Content-Type: application/json
body:{
    "user": "xxx",
    "limit": 10,
    "offet": 1,
    "sort": "desc"
    "ids": [
        "111",
        "222"
    ]
    ....
    ....
}'

// 一个单用注册接口
POST /user
```

现在我们发现列表查询方法和用户注册接口发生了冲突.这个就是下一个问题,参数中不能有动词这问题了,这个下一小结讨论.另外其实在日常使用中可能会把分页参数放到url中,
其他参数条件放到body中,这是不可取的方式,因为服务端处理参数需要对url参数进行解析,还有对body参数进行解析开发逻辑变多,代码量也增多,以及url中参数和body中参数冲突等问题

> 3. 参数中不能使用动词

先回顾一下动词有哪些,HTTP中规定的动词有以下几个

- GET
- POST
- UPDATE
- PATCH
- DELETE
- HEAD
- OPTIONS
- COPY
- LINK
- UNLINK
- PURGE
- LOCK
- UNLOCK
- PROPFIND
- VIEW

最常见的就是那6个GET POST PUT UPDATE DELETE HEAD OPTIONS.按照规范不能使用这些相关的动词,或者说隐含动词,比如POST请求对应的insert create
这是不允许的.现在按照上一问题的结尾所说来接着进行讨论。由于参数参数长度查询条件等问题,我们违反了规范更改了查询方式为post+json的形式。现在接口路由冲突,
由此现在我们不能完全遵守restful规范了,现在我们需要改路由规则,需要增加内容来进行区分。那具体改哪个接口呢。是列表查询还是注册接口呢,此种情况我个人认为可以改成如下方式:

```
// 列表查询
POST /user/list 
Content-Type: application/json
body:{
    "user": "xxx",
    "limit": 10,
    "offet": 1,
    "sort": "desc"
    "ids": [
        "111",
        "222"
    ]
    ....
    ....
}'

// 一个单用注册接口
POST /user
```

当然实际使用过程中比如复杂的管理后台ERP等路由规则要复杂的多,我们可以采用层级目录的形式加上一些动词来进行表示路由规则,比如以下示例

```
[GIN-debug] GET    /v1/user/overview/statistic --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] GET    /v1/user/overview/list    --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/user/chain/parallel/register --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/user/chain/parallel/detail --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/user/chain/parallel/list --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/user/chain/parallel/update/state --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/user/chain/parallel/delete --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] GET    /v1/user/chain/momson/list --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/user/chain/momson/upload/key --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/user/chain/momson/update/state --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/user/chain/momson/delete --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] GET    /v1/user/adapter/list     --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/user/adapter/register --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/user/adapter/update   --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/user/adapter/update/state --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/user/adapter/upload   --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] GET    /v1/user/tx/list          --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] GET    /v1/user/tx/status/list   --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] GET    /v1/user/tx/statistic     --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] GET    /v1/admin/overview/statistic --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] GET    /v1/admin/overview/list   --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] GET    /v1/admin/adapter/list    --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/admin/adapter/register --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/admin/adapter/update  --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/admin/adapter/update/state --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/admin/adapter/upload  --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/admin/chain/chlid/detail --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/admin/chain/chlid/list --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/admin/chain/chlid/update/state --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/admin/chain/chlid/delete --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/admin/chain/parallel/detail --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/admin/chain/parallel/list --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/admin/chain/parallel/update/state --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/admin/chain/parallel/delete --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/admin/chain/momson/detail --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/admin/chain/momson/upload/key --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/admin/chain/momson/update/state --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] POST   /v1/admin/chain/momson/delete --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] GET    /v1/admin/tx/list         --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] GET    /v1/admin/tx/status/list  --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)
[GIN-debug] GET    /v1/admin/tx/statistic    --> admin/pkg/net/http/middleware.Limit.func1 (3 handlers)

```

上述是摘自一个golang语言的web gin框架路由输出,这是一个管理后台,可分为两种权限的用户,普通用户,和管理员,然后该后台主要分为4种功能模块，
首页概览,链的管理,链的模块下还分子模块,有平行链,母子链,子链模块。适配器模块,交易列表模块

- 普通用户
    - 首页
    - 链管理模块
        - 母链模块
        - 子链模块
        - 平行链
    - 适配器模块
    - 交易列表模块
- 管理员
    - 首页
    - 链管理模块
        - 母链模块
        - 子链模块
        - 平行链
    - 适配器模块
    - 交易列表模块

其实说完大概的模块需求后,你直接读取路由内容大概就能猜出来这个接口是干什么的了,以及它所属的模块,以及作用域的划分。我所采用的路由方式其实就是层级目录的概念,
文件目录的概念,这样去理解可能会轻松很多,这种方式有什么好处,我人为在做权限控制的时候可以根据模块来进行划分,或者说层级目录划分,方便后端和前端权限组的控制，
比如根据路由前缀的方式来进行判断.谈到可读性的问题，我们可以列举restful的例子

```
/zoos/1/animals //id为1的动物园中的所有动物
/zoos/1;2;3 //id为1，2，3的动物园
/zoos/1/areas/3/animals/4 // 动物园区域id为1的,所有动物种类id为3,且动物为id为4的
```

以上这三种以及其它更复杂的场景就不列举了,本人认为代码可读性不高,不能明确分辨出请求中哪个是参数那个是资源限定条件,在测试使用中都稍麻烦一些。 当然还有一些问题,这个放到下一章节去讲。
ok回过头来上述是一个管理后台的大概区分,实际的ERP比这复杂的多,分国家,分城市地域,又分多种角色,此时restful该怎么表示这个路由规则？开发过ERP或者管理后台的人你一定能懂一些痛点。 下面是一些在[1]用户评论

```
BTQ 说:
一直不太明白，为什么要用http method来表示操作的类型，GET/PUT/POST/DELETE本身就不太好理解，表现力又有限。除了CURD之外，有时我们远程服务还得提供计算、校验等其他服务，决定用哪个method的时候，总感觉自己玩自己
```

```
xiaoc 说:
====
在RESTful架构中，每个网址代表一种资源（resource），所以网址中不能有动词，只能有名词，而且所用的名词往往与数据库的表格名对应。一般来说，数据库中的表都是同种记录的"集合"（collection），所以API中的名词也应该使用复数。
====

疑问

RESTful中的resource 和数据库中的table是一一对应的吗？ 如果是一一对应关系那么不就成了资源就是数据库中的数据了吗？ 难道数据库存储的时候一定要按照资源的结构来？ 而不是数据是数据，资源是资源？
我认为数据库中的数据结构只是为了存储资源的时候能够更方便，更快捷，更灵活而且可扩展等，但是资源是业务数据，他不一定就要和数据库中的某个table对应，可以使一个table中的一条数据，也可以是多个表数据的集合或者数据库表中的数据经过抽象加工后的一种数据。
```

```
石樱灯笼 说：

网址中不能有动词，只能有名词。那么如果我做一个登录功能，允许两种登录方式，用户名/密码 和 手机/验证码，那么只做一个login接口的话，我是需要在后端内部先判断post过来的data里是否有 用户密码或手机验证码，之后再判断变量合法性？感觉后端更臃肿了。
按照之前的旧思路，只要做两个接口 loginByPassword 和 loginByCaptcha 就好了，没有那么多if，代码也会简洁很多。
```

```
神冰 说：

弱弱的问下大师，支付宝开放平台的接口为什么没有遵循Restful，支付宝那么设计的目的是为了什么呢？
```

```
flynn 说：

有多少公司真正按RESTful来做？
```

```
飞流 说：

我也很疑惑，如果使用Restful风格，定义接口数量是有限的，比如：
GET /articles
GET /articles?published=true
GET /articles?title=标题1
GET /articles?pageNum=1&pageSize=10
使用SpringMVC会被mapping成一个接口，根本就不能这么定义
```

```
ticket 说：

REST感觉充其量就是一个url定义规范，有说这是什么restful架构，怎么上升到架构层次的，它分明对后端根本就是毫无约束也毫无指导意义呀？
而且项目稍微复杂一点，怎么可能只有对一个对象的增删改查？可能一个请求会涉及到无数个资源对象，这按restful该怎么设计？难道你能在url里面把这n个对象都列出来吗？
感觉这规范就是个没实际用处的东西，就是个url设计参考罢了。
```

> 4. url中有魔法变量

上一节简单说了一下url中有魔法变量的问题,其实还有以下其他方面的问题比如:

#### 1. 路由冲突问题

不同的语言以及框架的不同实现路由规则方式不同.有的使用前缀树比如radix tree等,更有甚至的直接使用map键值对的方式来实现。就以golang语言社区使用度最高的
web框架gin来说,在使用魔法变量这种方式,当层级过多,或者不规范,就会有路由冲突的问题出现。

#### 2. grpc路由方式支持不友好,以及其它路由转换方式

如今微服务大行其道,服务与服务之间通讯采用rpc通信的形式,在grpc中可以支持一http请求转换成rpc调用的形式,这个功能很好,在我们实际使用有些场景中,只写一个rpc 方法不用在写http服务了。
ok目前来说grpc中支持的路由规则比较简单,基本采用map key value的形式进行维护,对应复杂的路由规则支持不够友好，当然了技术上是可以补充上一些缺点。

#### 2. 可读性不高

这个没什么可说的前面也有提到,在多层级嵌套下可读性下降

上面说了那么魔法变量这种真的不可取么,也不是的,我个人认为,在实际使用场景中还是可以用但最好不要大于2个,越多越容易出现问题,最好用一个且这个使用场景是在根据id查询唯一数据的业务场景,
且魔法变量最好不要放到url中间的位置,为啥不放到中间位置,前面有提到,参数和资源界定分不清楚,例如：

```
// 我们定义的是这样的查询操作，查询某个指定动物园的指定动物
GET /zoos/ID/animals/ID
// 在我们实际业务中可能碰到的请求数据是这样的
GET /zoos/monkey/animals/monkey
```

先说下上述查询的意思.查询动物园id为monkey的且动物id为monkey的,当然了实际设计场景中id不会命名成这种单词的形式,但是一定会有这种类似业务员场景存在的。
那么现在一个用户测试人员或者联调开发者不能很好的分辨出资源限定以及参数问题,他不知道应该改哪个位置的参数,当然了实际过程中会有文档来进行参考了。

> 5. 使用单词复数的形式

有些博文介绍,或者规范说,采用单词复数的形式,我想说对于那些单词不好的人真的好么？比如`把y去掉加上es`的单词场景,我不确定了我去百度一下？ 在高强度时间迫切的情况下一线开发人员去抠字眼？
我认为没必要吧,只需要把单词拼对了知道啥意思,单词没有错误就ok了吧？更何况有的人因为失误或者英语不好连单词都打不对呢。

> 6. 在使用jwt等token校验时,敏感接口查询场景时id暴露存在的问题

前端和后端交互的敏感的业务场景,我们需要加入token验证,我们对数据查询等操作,都需要token比如我们使用jwt方式来进行验证,比如当前用户想查询自己的账户余额, 此时我们的查询不能在url中有敏感数据例如

```
GET user/wallet?uid=1
GET user/wallet/:id
```

这样会有什么问题？

1. 当url暴露之后很容易暴露具体某个用户的id,因为比较容易可读,如果有社工的人员或者黑客什么的会有些潜在安全问题,当然我的意思不是说不能在url中使用参数传递, 可以使用用一些不重要的字段,比如在分页查询中传入limit
   offset.

2.参数重复定义问题可能,比如jwt的场景下,payload中会存用户id等相关字段值,这些字段值层参数是包含签名运算的,所以不必担心id篡改的问题,因为token校验会不通过,
回到正题我想说的是,从token中就直接取id了就不用再url中或者body中取数据了,因为我可以伪造id来进行查询数据这样岂不是知道了其他用户id就可以查询别的用户数据了？

> 7.其他业务场景,比如跨域场景内 jsonp场景等

# 总结

本文诞生的目的不是否定restful设计！！！笔者只是从实际开发场景中进行探讨,我上面有很多都提到一些题外话,比如沟通成本测试操作可读性方面等问题,虽然看起来很无关紧要,
但是真的很重要,有些规范会束缚住我们开发效率,规范本来的目的就是为了统一协作达到更高的效率。另外我所说的一些要点以及解决方案不见的是对的,只是我个人的一个看法. 那么restful规范到底需不需要遵守？我认为是必须遵守,
但是得合理遵守,有些规范不适合所有的场景,世界上没有完美无缺的方案,只有适合的方案, 我父亲在小时的时候总是告诉我理论实践相结合.只有理论没有实践就是空中楼阁, 纸上谈兵,最近这几年领域驱动开发概念抄的非常火热,ppt做的一个比一个精彩,
但是能几个落实到实际的项目中，在实际开发中都是捉襟见肘,当然我没有说领域驱动就是无用的, 他还在实践和成长的道路上。得需要知行合一。

# 参考

[1] http://www.ruanyifeng.com/blog/2014/05/restful_api.html#comment-text

https://github.com/godruoyi/restful-api-specification

https://github.com/aisuhua/restful-api-design-references

2022年3月26日凌晨3:30

完~