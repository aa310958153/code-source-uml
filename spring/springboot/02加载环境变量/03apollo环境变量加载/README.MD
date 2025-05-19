# 简单使用
## 依赖引入
- 引入pom依赖
```xml
 <dependency>
    <groupId>com.ctrip.framework.apollo</groupId>
    <artifactId>apollo-client</artifactId>
    <version>1.3.0</version>
</dependency>
```
## spring boot方式
- jar包里面的Meta-Info会自动装配此类
```text
com.ctrip.framework.apollo.spring.boot.ApolloApplicationContextInitializer
->
com.ctrip.framework.apollo.internals.RemoteConfigRepository#loadApolloConfig
```
- 配置
```yml
#Apollo 配置
app:
  id: apollo-test                            #应用ID
apollo:
  cacheDir: /opt/data/                       #配置本地配置缓存目录
  cluster: default                           #指定使用哪个集群的配置
  meta: http://192.168.2.11:30002            #DEV环境配置中心地址
  autoUpdateInjectedSpringProperties: true   #是否开启 Spring 参数自动更新
  bootstrap:
    enabled: true                            #是否开启 Apollo
    namespaces: application.yml,bizconfig.yml    #设置 Namespace
    eagerLoad:
      enabled: false                         #将 Apollo 加载提到初始化日志系统之前  
```
## 非spring boot 方式
使用注解手动注入容器
```text
@EnableApolloConfig
```
## 动态修改原理
- 启动时缓存所有字段和set方法元数据信息
```text
com.ctrip.framework.apollo.spring.annotation.ApolloProcessor#postProcessBeforeInitialization
->
com.ctrip.framework.apollo.spring.annotation.ApolloProcessor#processField
com.ctrip.framework.apollo.spring.annotation.ApolloProcessor#processMethod
```
- 监听配置变化
```text[README.MD](README.MD)
com.ctrip.framework.apollo.internals.RemoteConfigLongPollService#doLongPollingRefresh 
-> 会发起请求这个地址 (长轮询)readTimeOut为90000 如果applo有修改则会返回200，否则返回304 304什么都不做，当返回200触发notify通知去拉取最新配置 可以理解为响应式的
http://10.4.3.218:9100/notifications/v2?cluster=default&appId=yxt-sentinel&ip=10.4.1.84&notifications=%5B%7B%22namespaceName%22%3A%22bizconfig.properties%22%2C%22notificationId%22%3A16553%7D%2C%7B%22namespaceName%22%3A%22application.yml%22%2C%22notificationId%22%3A16575%7D%5D
->触发通知
com.ctrip.framework.apollo.internals.RemoteConfigLongPollService#notify
->触发同步
com.ctrip.framework.apollo.internals.AbstractConfigRepository#trySync
->触发刷新
com.ctrip.framework.apollo.internals.LocalFileConfigRepository#onRepositoryChange
->写入本地文件
com.ctrip.framework.apollo.internals.LocalFileConfigRepository#persistLocalCacheFile
->根据缓存元数据写入应用
com.ctrip.framework.apollo.internals.AbstractConfigFile#onRepositoryChange
com.ctrip.framework.apollo.spring.property.AutoUpdateConfigChangeListener#onChange
```
- 监听非value方式可以采用这种方式， 
```java

  /**
     * 监听apollo配置，实现网关路由动态更新 这里统一发送事件com.ctrip.framework.apollo.internals.DefaultConfig#onRepositoryChange
     *
     * @param changeEvent 更新事件
     */
    @ApolloConfigChangeListener(value = "application.yml", interestedKeyPrefixes = "spring.cloud.gateway.")
    public void onChange(ConfigChangeEvent changeEvent) {
        refreshGatewayProperties(changeEvent);
    }
```
# 运维相关
- 默认缓存路径
```text
C:\opt\data\assist-core-toolkit\config-cache
```
