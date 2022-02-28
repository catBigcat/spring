# spring cloud nginx 

### 二、负载均衡策略

2.1 负载均策略

* 随机策略
* 线性策略
* 最小连接数量
* 相应时间权重
* 重试策略
* 可用过滤策略
* 区域过滤策略

2.2 手动配置

关闭 bibbon.eureka.enable: false

servername.nobbon.listOfServers: servers_name

2.3 rpc 请求超时配置

bibbon.connectTimeout 超时时间

​             .readTimeout 读取超时

可以加 servername来处理例如

servername.bibbon.connectTimeout 

2.4 重试机制



