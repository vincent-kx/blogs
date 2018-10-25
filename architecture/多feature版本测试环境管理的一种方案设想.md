# 多feature测试时后端测试环境管理的一种方案设想

## 背景

假如我们有一组后端服务为用户提供某些功能，各服务之间的调用关系（简化的）可能如下图

![简单的测试环境服务依赖关系图](/mycode/mygit/blogs/pics/simple_test_evn.png)

在实际的工作中，我们经常遇到这种场景：同时有多个feature在开发，比如feature_x,feature_y，feature_z，

这些feature的代码改动可能涉及到若干个相同或者不同的的后端服务，例如：

1）feature_x修改了app_srv0的<用户账户展示> 接口（假设该接口的名字是user_account_show），user_account_show接口依赖app_srv1的 <账户余额接口>user_account_balance

2）feature_y特性修改了user_account_balance接口，这个修改承诺接口的对外输出不改变，只是修改了用户的账户余额计算公式。

在feature_y测试通过之前”user_account_balance接口对外输出不改变“这个承诺是无法得到保障的，如果我们只有一个测试环境，多个feature都只能在这个测试环境上进行测试，那么feature_x,feature_y的测试只能串行。如果并行进行测试的话，服务部署会是下面这个样子的(浅绿表示feature_x，紫色表示feature_y)：

![并行测试](/mycode/mygit/blogs/pics/simple_test_env_1.png)

测试过程中如果feature_y有bug导致user_account_balance返回的数据不对，就会间接导致user_account_show接口返回的展示信息也不对，从而导致测试认为。但实际feature_x的修改。

## 常用的解决办法

1）测试串行，缺点：时间周期常，项目feature上线慢

2）并行部署多套测试环境，缺点：资源成本和维护成本都很高，如果服务众多或者并行开发的版本多那么基本不具备可执行性

3）使用一套环境强行并行测试，缺点：特性之间的相互影响可能导致多个特性的测试都需要返工，拖长各个feature测试的耗时

## 解决方案设想

先不废话，上个图

![多版本测试](/mycode/mygit/blogs/pics/multi_test_env.png)

由上面的分析我们可以知道要考虑的是“怎么已较小的代价实现测试环境特性隔离”，“代价小”也就是尽可能少的部署服务，只部署有修改的服务即可；“特性隔离”就是不同特性分支的请求分别路由到支持对应特性的服务版本上。参考上图：

feature_x修改了app_srv1，app_srv4，app_srv6，分别得到新版本服务app_srv1_x，app_srv4\_x，app_srv6\_x

feature_y修改了app_srv4，app_srv7，分别得到新版本服务app_srv4_y，app_srv7\_y

假设feature_x版本的请求为Rx，feature_y版本的请求为Ry，基线版本的请求为R0，只要做到Rx，Ry请求分别走不同的调用链即可保证feature_x和feature_y的测试相互不影响，也就是：

Rx的调用链：app_srv0 —> app_srv1_x —> app_srv2 —> app_srv4\_x —> app_srv6\_x —> app_srv7

Ry的调用链：app_srv0 —> app_srv1 —> app_srv2 —> app_srv4\_y —> app_srv6 —> app_srv7\_y

##如何做到这一点

 __让请求携带路由信息__(以下格式为非标准json格式，只是类似格式方便展示而已)

在请求的公共头部携带一个版本绑定的路由信息，比如编号feature_x\_20180101，feature_y_20180101

```json
//feature_x
comm_head:{
  route_id:feature_x_20180101
}

//feature_y
comm_head:{
  route_id:feature_y_20180101
}
```

存储一张路由表

```json
route_table:{
  	feature_x_20180101:{
      [
        appid : app_srv1,
        srv_version : app_srv1_xxx,
        addr:"ip:port"
      ],
      [
        appid : app_srv4,
        srv_version : app_srv4_xxx,
        addr:"ip:port"
      ],
      [
        appid : app_srv1,
        srv_version : app_srv6_xxx,
        addr:"ip:port"
      ],
    }
  	feature_y_20180101:{
      [
        appid : app_srv4,
        srv_version : app_srv4_yyy,
        addr:"ip:port"
      ],
      [
        appid : app_srv7,
        srv_version : app_srv7_yyy,
        addr:"ip:port"
      ]
    }
}
```

每个服务维护自己的appid，发起rpc的服务通过某种方式获取路由表配置，从获取的路由表中寻找映射的appid，和被调用目标appid（调用方一定知道被掉方的appid）进行对比，找到需要调用的服务地址和端口，例如：

feature_x的请求Rx到达app_srv0之后，app_srv0需要调用app_srv1，app_srv0解析出请求头部（comm_head）的route_id：feature_x_20180101，使用route\_id获取到路由表feature_x\_20180101项下面的所有配置，从获得的配置中寻找appid=app_srv1的项（这里服务的数量是很有限的，不必担心遍历的性能），获取对应的ip和端口发起调用，这样就可以是Rx请求触发的第一次rpc调用的是feature_x版本的app_srv1\_x而不是基线版本的app_srv1或者feature_y版本的app_srv1\_y，如果找不到对应的想就采用默认路由调用，也就是调用基线版本；发起调用的时候透传请求头里面的route_id，这样后续调用链的每个服务都能使用上述过程找到请求要调用的版本分支，这里需要增加的就是对路由表的维护工作以及客户端发起请求的时候需要增加请求头的写入。发布上线之后也可以采用这种方法进行AB test，如果不需要可以对配置进行修改或者前端发起请求的时候不携带路由信息。

## 路由表的管理和维护

1）交给mesh

2）服务发布的时候通过注册中心注册自己版本的路由信息，通过注册中心下发到各服务所在的机器，服务框架通过agent从本地读取配置



自己的一点思考，欢迎更多交流！！！



