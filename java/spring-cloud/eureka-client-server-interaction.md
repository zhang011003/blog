# Eureka客户端和服务端交互步骤

## 注册

Eureka客户端将运行信息注册到Eureka服务端。注册在第一次心跳时发生（30秒后）

## 续约

Eureka客户端需要每隔30秒通过心跳方式进行续约。续约的目的就是通知Eureka服务端当前实例还存活。如果服务端90秒内没有收到续约请求，则会将客户端实例从注册信息中移除。建议最好不要修改续约间隔，因为服务端会通过该信息判断是否和客户端通信有大范围的问题存在。

## 获取注册信息

Eureka客户端会从服务端获取注册信息并缓存到本地。之后会使用该信息查找其它的服务。注册信息会比较上次获取周期和本次获取周期的差值更新（delta updates），然后进行周期性更新（每30秒一次）。差值信息在服务端会保存较长时间（大概3分钟），因为差值获取可能获取到相同的实例信息。Eureka客户端会自动处理重复信息。

当Eureka客户端获取到差异信息后，与从服务端取回的实例数进行比较，并对信息进行合并。如果某些情况下信息不匹配，会重新获取整个注册信息。Eureka服务端缓存了差异的负载、全部注册信息以及每个应用的注册信息，压缩和未压缩的都有。负载支持 JSON/XML 格式。Eureka客户端通过Jersey获取压缩的JSON格式信息。

## 注销

当Eureka客户端关闭时，会发送注销请求到服务端。这样会从服务端注册信息中将该实例删除。

## 时间延期

Eureka客户端的所有操作都会在一定时间后在服务端以及其它客户端上体现。这是因为服务端的负载缓存是周期性刷新的。Eureka客户端也会周期性获取变更信息。因此所有客户端都知道变更信息可能要花费2分钟时间。

## Eureka提供的Rest接口

对应的类`org/springframework/cloud/netflix/eureka/http/RestTemplateEurekaHttpClient.java`

下面的 **appID**  为应用名称， **instanceID**  为相关实例的唯一id。

对于 JSON/XML ， content types 必须为 **application/xml** 或 **application/json** 

| HTTP地址                                                     | 含义                                                         | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| POST /eureka/apps/**appID**                                  | 注册新应用实例                                               | 输入：JSON/XML。成功则返回HTTP代码204                        |
| DELETE /eureka/apps/**appID**/**instanceID**                 | 注销应用实例                                                 | 成功则返回HTTP代码200                                        |
| PUT /eureka/apps/**appID**/**instanceID**                    | 发送应用实例心跳                                             | 成功则返回HTTP代码200，如果**instanceID**不存在，则返回404   |
| GET /eureka/apps                                             | 查询所有实例                                                 | 成功则返回HTTP代码200。输出：JSON/XML                        |
| GET /eureka/apps/**appID**                                   | 查询**appID**的所有实例                                      | 成功则返回HTTP代码200。输出：JSON/XML                        |
| GET /eureka/apps/**appID**/**instanceID**                    | 查询**appID**/**instanceID**的实例信息                       | 成功则返回HTTP代码200。输出：JSON/XML                        |
| GET /eureka/instances/**instanceID**                         | 查询指定**instanceID**的实例信息                             | 成功则返回HTTP代码200。输出：JSON/XML                        |
| PUT /eureka/apps/**appID**/**instanceID**/status?value=OUT_OF_SERVICE | 状态更新。将指定**appID**/**instanceID**的实例状态设置为out of service | 成功则返回HTTP代码200，失败返回500                           |
| DELETE /eureka/apps/**appID**/**instanceID**/status?value=UP（value=UP可选） | 将实例放回服务（移除覆盖）                                   | 成功则返回HTTP代码200，失败返回500                           |
| PUT /eureka/apps/**appID**/**instanceID**/metadata?key=value | 更新metadata                                                 | 成功则返回HTTP代码200，失败返回500                           |
| GET /eureka/vips/**vipAddress**                              | 查询指定**vip address**下的所有实例                          | 成功则返回HTTP代码200，输出：JSON/XML。如果**vipAddress**不存在，返回404 |
| GET /eureka/svips/**svipAddress**                            | 查询指定**secure vip address**下的所有实例                   | 成功则返回HTTP代码200，输出：JSON/XML。如果**svipAddress**不存在，返回404 |



## 参考
> [Understanding eureka client server communication](https://github.com/Netflix/eureka/wiki/Understanding-eureka-client-server-communication)
>
> [Eureka REST operations](https://github.com/Netflix/eureka/wiki/Eureka-REST-operations)
>
> 


