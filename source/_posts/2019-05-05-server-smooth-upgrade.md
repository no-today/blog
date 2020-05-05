---
title: Spring Cloud 平滑升级
date: 2019-05-05

categories:
    - Developer
---

降低服务不可用时长

<!--more-->

## CAP定理

- 一致性（Consistency）：等同于所有节点访问同一份最新的数据副本
- 可用性（Availability）：每次请求都能获取到非错的响应，但是不保证获取的数据为最新数据
- 分区容错性（Partition tolerance）：以实际效果而言，分区相当于对通信的时限要求。系统如果不能在时限内达成数据一致性，就意味着发生了分区的情况，必须就当前操作在C和A之间做出选择

### Zookeeper

ZK 的设计原则是 CP，即强一致性和分区容错性。它保证数据的强一致性，但舍弃了可用性。如果出现网络问题可能会影响 ZK 但选举，导致 ZK 注册中心的不可用。

### Eureka & Nacos

Eureka 的设计原则是 AP，即可用性和分区容错性。它保证了注册中心的可用性，但舍弃了数据一致性，各节点上但数据有可能是会不一致但（会最终一致）。

### 差异

ZK 是注册中心跟服务之间建立长链接来建立联系的，服务提供者挂了注册中心会实时感知到，实时推送最新的注册信息给服务消费者。

Eureka 和 Nacos 这些 Spring Cloud 系的注册中心都是用定时任务 http 轮训去定时汇报、拉取信息的，服务提供者挂了，服务消费者会持有一段时间的脏数据，直到定时任务下一次去注册中心拉取最新的注册信息。

## Ureka 服务调用关系

### 服务提供者

1. 服务启动后向注册中心发起 register 请求，注册服务。
2. 在运行过程中定时向注册中心发送 renew 心跳，证明 "我还活着"。
3. 非暴力停止服务时向注册中心发起 cancel 请求，清空当前服务注册信息。

### 服务消费者

1. 服务启动后从注册中心拉取服务注册信息。
2. 在运行过程中定时任务去注册中心拉取最新的服务注册信息。
3. 服务消费者发起远程调用:
    - 服务消费者（北京）会从服务注册信息中选择同机房的服务提供者（北京），发起远程调用。只有同机房的服务提供者挂了才会选择其他机房的服务提供者（青岛）。
    - 服务消费者（天津）因为同机房内没有服务提供者，则会按照负载均衡算法选择北京或者青岛的服务提供者，发起远程调用。

## 服务分批升级

先发布一批做验证，发现问题立即回滚，没问题再发布下一批，直到全部节点发布完毕。

## 服务不可用

数据不一致导致远程调用请求到了已被停止的服务。

即使旧服务进程被正常停止，向注册中心发送了 cancel 请求，让注册中心清空了自己的注册信息。其他服务消费者还是会持有旧的注册信息，直到下一次定时任务执行去注册中心拉取最新的注册信息，不采取措施的话这期间产生的远程调用还是会打到已停止的服务去，疯狂报 500。

## 降低不可用时长

### 调小轮训拉取注册信息的时间

无论如何我们都不可能达到强数据一致性(除非放弃可用性)，只能尽可能的降低不可用时长。

这些值不能太大也不能太小, 如果该值太大，则很可能将流量转发过去的时候，该 instance 已经不存活了。如果该值设置太小了，则 instance 则很可能因为临时的网络抖动而被摘除掉。

下面只保留了我们需要关注的配置，根据实际场景调整参数值。

### Eureka

```yaml
eureka:
  client:
    instance-info-replication-interval-seconds: 10  # 间隔多久更新实例信息到注册中心
    registry-fetch-interval-seconds: 10             # 间隔多久去注册中心拉取注册信息(默认30秒)
  instance:
    lease-renewal-interval-in-seconds: 5            # 向注册中心发生心跳的频率(默认30秒)
    lease-expiration-duration-in-seconds: 10        # 注册中心在接收到上一个心跳之后等待的时间, 超过该时间会移除实例，流量不会转发过去了。
```

### Nacos

```yaml
spring:
  cloud:
    nacos:
      discovery:
        metadata:
          heart-beat-interval: 3    # 心跳包发送周期,单位为秒
          heart-beat-timeout: 6     # 心跳超时时间,即服务端6秒收不到心跳,会将客户端注册的实例设为不健康
          ip-delete-timeout: 9      # 实例删除的超时时间,即服务端9秒收不到客户端心跳,会将客户端注册的实例删除
```

### 调整 Feign 重试机制

```yaml
feign:
  hystrix:
    enabled: true
  client:
    config:
      default:
        # 对当前实例的重试次数，放弃失败的节点，转而重试其他节点。
        maxAutoRetries: 0
        # 切换实例的重试次数
        maxAutoRetriesNextServer: 3
        okToRetryOnAllOperations: true
        connectTimeout: 5000
        readTimeout: 5000
```

## 参考资料

- [微服务注册中心 Eureka 架构深入解读](https://www.infoq.cn/article/jlDJQ*3wtN2PcqTDyokh)
- [Nacos 1.1.0发布，支持灰度配置和地址服务器模式](https://nacos.io/zh-cn/blog/nacos%201.1.0.html)
- [Nacos Issues #2064](https://github.com/alibaba/nacos/issues/2064)
- [Spring Cloud Eureka 服务实现不停机（Zero-downtime）部署](https://segmentfault.com/a/1190000022134014)
- [Feign Configuration](~/.m2/repository/org/springframework/cloud/spring-cloud-openfeign-core/2.1.3.RELEASE/spring-cloud-openfeign-core-2.1.3.RELEASE.jar!/META-INF/spring-configuration-metadata.json)
- [Hystrix Configuration](https://github.com/Netflix/Hystrix/wiki/Configuration)