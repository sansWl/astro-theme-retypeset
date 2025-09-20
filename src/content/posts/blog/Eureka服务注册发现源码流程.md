---
 title: Eureka服务注册发现源码流程简析
 published: 2025-03-11
 tags:
  - BlogPost
 lang: zh
 abbrlink:  f14c8695-a3e9 
---

### 一： 服务的注册
> 客户端通过执行InstanceInfoReplicator#run()调用DiscoveryClient#register()发送http请求进行注册 
> `InstanceInfoReplicator` 是更新同步当前服务到服务端的任务实现 
> //A task for updating and replicating the local instanceinfo to the remote server. 

![](https://img2024.cnblogs.com/blog/3426265/202503/3426265-20250311142333132-1751032344.png)

```java
//服务注册
boolean register() throws Throwable {
        logger.info("DiscoveryClient_{}: registering service...", this.appPathIdentifier);

        EurekaHttpResponse httpResponse;
        try {
            httpResponse = this.eurekaTransport.registrationClient.register(this.instanceInfo);
        } catch (Exception var3) {
            Exception e = var3;
            logger.warn("DiscoveryClient_{} - registration failed {}", new Object[]{this.appPathIdentifier, e.getMessage(), e});
            throw e;
        }

        if (logger.isInfoEnabled()) {
            logger.info("DiscoveryClient_{} - registration status: {}", this.appPathIdentifier, httpResponse.getStatusCode());
        }

        return httpResponse.getStatusCode() == Status.NO_CONTENT.getStatusCode();
    }
//服务续约
/**
eureka 初始化定时任务，依据设定的心跳时间触发 renew方法，检测服务是否宕机
*/
    boolean renew() {
        try {
            EurekaHttpResponse<InstanceInfo> httpResponse = this.eurekaTransport.registrationClient.sendHeartBeat(this.instanceInfo.getAppName(), this.instanceInfo.getId(), this.instanceInfo, (InstanceInfo.InstanceStatus)null);
            logger.debug("DiscoveryClient_{} - Heartbeat status: {}", this.appPathIdentifier, httpResponse.getStatusCode());
            if (httpResponse.getStatusCode() == Status.NOT_FOUND.getStatusCode()) {
                this.REREGISTER_COUNTER.increment();
                logger.info("DiscoveryClient_{} - Re-registering apps/{}", this.appPathIdentifier, this.instanceInfo.getAppName());
                long timestamp = this.instanceInfo.setIsDirtyWithTime();
                boolean success = this.register();
                if (success) {
                    this.instanceInfo.unsetIsDirty(timestamp);
                }

                return success;
            } else {
                return httpResponse.getStatusCode() == Status.OK.getStatusCode();
            }
        } catch (Throwable var5) {
            Throwable e = var5;
            logger.error("DiscoveryClient_{} - was unable to send heartbeat!", this.appPathIdentifier, e);
            return false;
        }
    }
```
> 服务端服务注册接受和存储 

![](https://img2024.cnblogs.com/blog/3426265/202503/3426265-20250311143033813-575040911.png)

```java
//eureka 客户端会通过此方法注册保存到eureka server 的内存中
public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
        this.read.lock();

        try {
            Map<String, Lease<InstanceInfo>> gMap = (Map)this.registry.get(registrant.getAppName());
            EurekaMonitors.REGISTER.increment(isReplication);
            if (gMap == null) {
                ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap();
                gMap = (Map)this.registry.putIfAbsent(registrant.getAppName(), gNewMap);
                if (gMap == null) {
                    gMap = gNewMap;
                }
            }

            Lease<InstanceInfo> existingLease = (Lease)((Map)gMap).get(registrant.getId());
            if (existingLease != null && existingLease.getHolder() != null) {
                Long existingLastDirtyTimestamp = ((InstanceInfo)existingLease.getHolder()).getLastDirtyTimestamp();
                Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
                logger.debug("Existing lease found (existing={}, provided={}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
                if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
                    logger.warn("There is an existing lease and the existing lease's dirty timestamp {} is greater than the one that is being registered {}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
                    logger.warn("Using the existing instanceInfo instead of the new instanceInfo as the registrant");
                    registrant = (InstanceInfo)existingLease.getHolder();
                }
            } else {
                synchronized(this.lock) {
                    if (this.expectedNumberOfClientsSendingRenews > 0) {
                        ++this.expectedNumberOfClientsSendingRenews;
                        this.updateRenewsPerMinThreshold();
                    }
                }

                logger.debug("No previous lease information found; it is new registration");
            }

            Lease<InstanceInfo> lease = new Lease(registrant, leaseDuration);
            if (existingLease != null) {
                lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
            }

            ((Map)gMap).put(registrant.getId(), lease);
            this.recentRegisteredQueue.add(new Pair(System.currentTimeMillis(), registrant.getAppName() + "(" + registrant.getId() + ")"));
            if (!InstanceStatus.UNKNOWN.equals(registrant.getOverriddenStatus())) {
                logger.debug("Found overridden status {} for instance {}. Checking to see if needs to be add to the overrides", registrant.getOverriddenStatus(), registrant.getId());
                if (!this.overriddenInstanceStatusMap.containsKey(registrant.getId())) {
                    logger.info("Not found overridden id {} and hence adding it", registrant.getId());
                    this.overriddenInstanceStatusMap.put(registrant.getId(), registrant.getOverriddenStatus());
                }
            }

            InstanceInfo.InstanceStatus overriddenStatusFromMap = (InstanceInfo.InstanceStatus)this.overriddenInstanceStatusMap.get(registrant.getId());
            if (overriddenStatusFromMap != null) {
                logger.info("Storing overridden status {} from map", overriddenStatusFromMap);
                registrant.setOverriddenStatus(overriddenStatusFromMap);
            }

            InstanceInfo.InstanceStatus overriddenInstanceStatus = this.getOverriddenInstanceStatus(registrant, existingLease, isReplication);
            registrant.setStatusWithoutDirty(overriddenInstanceStatus);
            if (InstanceStatus.UP.equals(registrant.getStatus())) {
                lease.serviceUp();
            }

            registrant.setActionType(ActionType.ADDED);
            this.recentlyChangedQueue.add(new RecentlyChangedItem(lease));
            registrant.setLastUpdatedTimestamp();
            this.invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
            logger.info("Registered instance {}/{} with status {} (replication={})", new Object[]{registrant.getAppName(), registrant.getId(), registrant.getStatus(), isReplication});
        } finally {
            this.read.unlock();
        }

    }
```
> 如图，registry保存有将注册、已注册到server 的eureka客户端 instance信息；
> 将要注册到注册表registry中的instance，会创建一个map结构保存


![](https://img2024.cnblogs.com/blog/3426265/202503/3426265-20250311135621493-58076025.png)

### 二： 服务的发现
> 依据上图展示，服务注册采取的是客户端创立DiscoveryClient建立http请求，同理利用此DiscoveryClient实例通过请求完成服务的发现，不再赘述；
> 重点将放入服务发现的缓存和调用

`1.`
```java
//其他发现实现 DiscoveryClient#getAndUpdateDelta 包括更新等具体操作不作继续深入讨论
    /**
     * Gets the full registry information from the eureka server and stores it locally.
     * When applying the full registry, the following flow is observed:
     *
     * if (update generation have not advanced (due to another thread))
     *   atomically set the registry to the new registry
     * fi
     *
     * @return the full registry information.
     * @throws Throwable
     *             on error.
     */
    private void getAndStoreFullRegistry() throws Throwable {
        long currentUpdateGeneration = fetchRegistryGeneration.get();

        logger.info("Getting all instance registry info from the eureka server");

        Applications apps = null;
        EurekaHttpResponse<Applications> httpResponse = clientConfig.getRegistryRefreshSingleVipAddress() == null
                ? eurekaTransport.queryClient.getApplications(remoteRegionsRef.get())
                : eurekaTransport.queryClient.getVip(clientConfig.getRegistryRefreshSingleVipAddress(), remoteRegionsRef.get());
        if (httpResponse.getStatusCode() == Status.OK.getStatusCode()) {
            apps = httpResponse.getEntity();
        }
        logger.info("The response status is {}", httpResponse.getStatusCode());

        if (apps == null) {
            logger.error("The application is null for some reason. Not storing this information");
        } else if (fetchRegistryGeneration.compareAndSet(currentUpdateGeneration, currentUpdateGeneration + 1)) {
    //存入本地服务缓存，即客户端将server端的服务注册信息缓存了一份，存在后续的缓存更新机制不做深入
    //this.filterAndShuffle(apps) 目的是处理 UP 状态的实例以及 打乱保证随机性 
    // Shuffling helps in randomizing the applications list there by avoiding the same instances receiving traffic during start ups.
            localRegionApps.set(this.filterAndShuffle(apps));
            logger.debug("Got full registry with apps hashcode {}", apps.getAppsHashCode());
        } else {
            logger.warn("Not updating applications as another thread is updating it already");
        }
    }
```
 `2.` 调用
 >  一： 利用DiscoveryClient获取实例的信息，再构建http请求
 >  二： 使用组件fegin完成服务转发
 ```java
@FeignClient(value = "eurekaclient")
public interface ApiService {

    @RequestMapping(value = "/index",method = RequestMethod.GET)
    String index();
}

/**
等价于 new httpclient(eurekaClient) => 发送 /index 接口并接受到response
*/
 ```