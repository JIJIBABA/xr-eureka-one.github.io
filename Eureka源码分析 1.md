## Eureka源码分析 1

### 背景

该篇文章主要介绍自我保护的选择，eureka的3级缓存原理，剔除时间间隔的设定等问题。涉及到的相关配置如下。



```java
  server:
    #自我保护
    enable-self-preservation: false
    #保护因子    
    renewal-percent-threshold: 0.85
    #剔除时间间隔    
    eviction-interval-timer-in-ms: 1000  
    #是否使用readOnly缓存   
    use-read-only-response-cache: false #
    #readWriteCacheMap 和 readOnly  
    response-cache-update-interval-ms: 1000
```

首先我们要知道EurekaServerAutoConfiguration 引用了 EurekaServerInitializerConfiguration， EurekaServerInitializerConfiguration实现了 SmartLifecycle 里面有个start方法

因此就会eureka启动的时候就会执行start方法 

```java
//下面的代码就是我们本次讲解源码的入口
public void start() {
        (new Thread(() -> {
            try {
                this.eurekaServerBootstrap.contextInitialized(this.servletContext); //点击进入
                log.info("Started Eureka Server");
                this.publish(new EurekaRegistryAvailableEvent(this.getEurekaServerConfig()));
                this.running = true;
                this.publish(new EurekaServerStartedEvent(this.getEurekaServerConfig()));
            } catch (Exception var2) {
                log.error("Could not initialize Eureka servlet context", var2);
            }

        })).start();
    }
//className :EurekaServerInitializerConfiguration
```



### 自我保护

#### 自我保护源码

```java
//初始化上下文
protected void initEurekaServerContext() throws Exception {
        JsonXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(), 10000);
        XmlXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(), 10000);
        if (this.isAws(this.applicationInfoManager.getInfo())) {
            this.awsBinder = new AwsBinderDelegate(this.eurekaServerConfig, this.eurekaClientConfig, this.registry, this.applicationInfoManager);
            this.awsBinder.start();
        }

        EurekaServerContextHolder.initialize(this.serverContext);
        log.info("Initialized server context");
        int registryCount = this.registry.syncUp();//拉取注册信息
        this.registry.openForTraffic(this.applicationInfoManager, registryCount); //点击进入
        EurekaMonitors.registerAllStats();
    }
//className :EurekaServerBootstrap
```



```java
public void openForTraffic(ApplicationInfoManager applicationInfoManager, int count) {
        this.expectedNumberOfClientsSendingRenews = count;//更新每次续约的次数
        this.updateRenewsPerMinThreshold(); //更新每次续约的预值
        logger.info("Got {} instances from neighboring DS node", count);
        logger.info("Renew threshold is: {}", this.numberOfRenewsPerMinThreshold);
        this.startupTime = System.currentTimeMillis();
        if (count > 0) {
            this.peerInstancesTransferEmptyOnStartup = false;
        }

        Name selfName = applicationInfoManager.getInfo().getDataCenterInfo().getName();
        boolean isAws = Name.Amazon == selfName;
        if (isAws && this.serverConfig.shouldPrimeAwsReplicaConnections()) {
            logger.info("Priming AWS connections for all replicas..");
            this.primeAwsReplicas(applicationInfoManager);
        }

        logger.info("Changing status to UP");
        applicationInfoManager.setInstanceStatus(InstanceStatus.UP);
        super.postInit(); //点击进入
    }
//类名：PeerAwareInstanceRegistryImpl
```



```java
    protected void postInit() {
        this.renewsLastMin.start();
        if (this.evictionTaskRef.get() != null) {
            ((AbstractInstanceRegistry.EvictionTask)this.evictionTaskRef.get()).cancel();
        }

        this.evictionTaskRef.set(new AbstractInstanceRegistry.EvictionTask());//设置一个定时任务
        this.evictionTimer.schedule((TimerTask)this.evictionTaskRef.get(), this.serverConfig.getEvictionIntervalTimerInMs(), this.serverConfig.getEvictionIntervalTimerInMs());
    }
//类名 ：AbstractInstanceRegistry
```

```java
 class EvictionTask extends TimerTask {
        private final AtomicLong lastExecutionNanosRef = new AtomicLong(0L);

        EvictionTask() {
        }

        public void run() {
            try {
                long compensationTimeMs = this.getCompensationTimeMs();
                AbstractInstanceRegistry.logger.info("Running the evict task with compensationTime {}ms", compensationTimeMs);
                AbstractInstanceRegistry.this.evict(compensationTimeMs); //看这个方法 剔除任务
            } catch (Throwable var3) {
                AbstractInstanceRegistry.logger.error("Could not run the evict task", var3);
            }

        }
```

```java
 //下面就是剔除任务的具体逻辑 
public void evict(long additionalLeaseMs) {
        logger.debug("Running the evict task");
        if (!this.isLeaseExpirationEnabled()) {
            logger.debug("DS: lease expiration is currently disabled.");
        } else {
            List<Lease<InstanceInfo>> expiredLeases = new ArrayList();
            Iterator var4 = this.registry.entrySet().iterator();

            while(true) {
                Map leaseMap;
                do {
                    if (!var4.hasNext()) {
                        int registrySize = (int)this.getLocalRegistrySize();//获取注册表的大小
                        int registrySizeThreshold = (int)((double)registrySize * this.serverConfig.getRenewalPercentThreshold()); //注册表大小 * 预测百分比（默认是0.85）
                        int evictionLimit = registrySize - registrySizeThreshold; //要剔除的数量（可以挂几个服务）
                        int toEvict = Math.min(expiredLeases.size(), evictionLimit);//取实际挂的服务跟 预期值的最小值
                        //上面的代码和自我保护有关 假如你有20个服务 （20*0.85 =17 个，意味着你可以挂3个服务，当挂第四个的时候就会触发自我保护 如果第四个服务真挂了 而不进行删除 会把有的请求路由到不可用的服务上
                        if (toEvict > 0) {
                            logger.info("Evicting {} items (expired={}, evictionLimit={})", new Object[]{toEvict, expiredLeases.size(), evictionLimit});
                            Random random = new Random(System.currentTimeMillis());

                            for(int i = 0; i < toEvict; ++i) {
                                int next = i + random.nextInt(expiredLeases.size() - i);
                                Collections.swap(expiredLeases, i, next);
                                Lease<InstanceInfo> lease = (Lease)expiredLeases.get(i);
                                String appName = ((InstanceInfo)lease.getHolder()).getAppName();
                                String id = ((InstanceInfo)lease.getHolder()).getId();
                                EurekaMonitors.EXPIRED.increment();
                                logger.warn("DS: Registry: expired lease for {}/{}", appName, id);
                                this.internalCancel(appName, id, false);
                            }
                        }

                        return;
                    }

                    Entry<String, Map<String, Lease<InstanceInfo>>> groupEntry = (Entry)var4.next();
                    leaseMap = (Map)groupEntry.getValue();
                } while(leaseMap == null);

                Iterator var7 = leaseMap.entrySet().iterator();

                while(var7.hasNext()) {
                    Entry<String, Lease<InstanceInfo>> leaseEntry = (Entry)var7.next();
                    Lease<InstanceInfo> lease = (Lease)leaseEntry.getValue();
                    if (lease.isExpired(additionalLeaseMs) && lease.getHolder() != null) {
                        expiredLeases.add(lease);
                    }
                }
            }
        }
    }
```

自我保护：正常情况下，当eureka接受不到服务的的心跳时就会进行服务剔除。但是当开启自我保护时

挂机数比率 >=1- (renewal-percent-threshold: 0.85) 时 就会触发自我保护不对服务进行剔除。

#### 自我保护利弊

1.当关闭自我保护后，如果服务心跳的终止是由于网络抖动所造成的，这样进行服务剔除会把好的服务进行删除，遭成好的服务不可用的情况。

2.如果开启自我保护，那么如果一个服务确实不可用，那么在调用方拉去服务列表时会拉去到不可用的服务。

#### 自我保护的选择

大家在实际应用时可以根据自己实际服务的规模进行选择。如果你允许挂的服务数一定（假如你允许挂10个服务），那么建议在服务总数少的时候不开自我保护，在服务多的情况下开自我保护。因为当你服务很多，进行多个机房部署，那么网络抖动造成收不到心跳的情况很正常，这时建议打开自我保护。

```java
    public void start() {
        (new Thread(() -> {
            try {
                this.eurekaServerBootstrap.contextInitialized(this.servletContext);
                log.info("Started Eureka Server");
                this.publish(new EurekaRegistryAvailableEvent(this.getEurekaServerConfig()));
                this.running = true;
                //发布服务启动事件，相关的还有服务挂掉的事件，也可以监控如果服务挂掉 发送邮件
                this.publish(new EurekaServerStartedEvent(this.getEurekaServerConfig()));
            } catch (Exception var2) {
                log.error("Could not initialize Eureka servlet context", var2);
            }

        })).start();
    }
//类EurekaServerInitializerConfiguration

//针对服务下线我们还可以通过邮件监控
  public void listen(EurekaInstanceCanceledEvent event) {
        //发送邮件 进行提醒

    }

```

还可以通过减小服务剔除的时间间隔来避免拉取的不可用服务

```java
//eviction-interval-timer-in-ms: 1000    设定服务剔除时间
protected void postInit() {
        this.renewsLastMin.start();
        if (this.evictionTaskRef.get() != null) {
            ((AbstractInstanceRegistry.EvictionTask)this.evictionTaskRef.get()).cancel();
        }

        this.evictionTaskRef.set(new AbstractInstanceRegistry.EvictionTask());
       //定期将没有心跳的服务剔除
        this.evictionTimer.schedule((TimerTask)this.evictionTaskRef.get(), 
                                    //剔除的时间间隔
                                    this.serverConfig.getEvictionIntervalTimerInMs(), this.serverConfig.getEvictionIntervalTimerInMs());
    }
//className :AbstractInstanceRegistry
```

#### 三级缓存

#### 相关源码

```java
//当服务向eureka中注册时会走下面的代码 如果想debug  启动eureka服务端，然后启动client端，就会进入下面的代码
@POST
    @Consumes({"application/json", "application/xml"})
    public Response addInstance(InstanceInfo info, @HeaderParam("x-netflix-discovery-replication") String isReplication) {
        logger.debug("Registering instance {} (replication={})", info.getId(), isReplication);
        if (this.isBlank(info.getId())) {
            return Response.status(400).entity("Missing instanceId").build();
        } else if (this.isBlank(info.getHostName())) {
            return Response.status(400).entity("Missing hostname").build();
        } else if (this.isBlank(info.getIPAddr())) {
            return Response.status(400).entity("Missing ip address").build();
        } else if (this.isBlank(info.getAppName())) {
            return Response.status(400).entity("Missing appName").build();
        } else if (!this.appName.equals(info.getAppName())) {
            return Response.status(400).entity("Mismatched appName, expecting " + this.appName + " but was " + info.getAppName()).build();
        } else if (info.getDataCenterInfo() == null) {
            return Response.status(400).entity("Missing dataCenterInfo").build();
        } else if (info.getDataCenterInfo().getName() == null) {
            return Response.status(400).entity("Missing dataCenterInfo Name").build();
        } else {
            DataCenterInfo dataCenterInfo = info.getDataCenterInfo();
            if (dataCenterInfo instanceof UniqueIdentifier) {
                String dataCenterInfoId = ((UniqueIdentifier)dataCenterInfo).getId();
                if (this.isBlank(dataCenterInfoId)) {
                    boolean experimental = "true".equalsIgnoreCase(this.serverConfig.getExperimental("registration.validation.dataCenterInfoId"));
                    if (experimental) {
                        String entity = "DataCenterInfo of type " + dataCenterInfo.getClass() + " must contain a valid id";
                        return Response.status(400).entity(entity).build();
                    }

                    if (dataCenterInfo instanceof AmazonInfo) {
                        AmazonInfo amazonInfo = (AmazonInfo)dataCenterInfo;
                        String effectiveId = amazonInfo.get(MetaDataKey.instanceId);
                        if (effectiveId == null) {
                            amazonInfo.getMetadata().put(MetaDataKey.instanceId.getName(), info.getId());
                        }
                    } else {
                        logger.warn("Registering DataCenterInfo of type {} without an appropriate id", dataCenterInfo.getClass());
                    }
                }
            }

            this.registry.register(info, "true".equals(isReplication));//点击进入
            return Response.status(204).build();
        }
    }
//className :ApplicationResource
```

```java
 //更新注册表  
public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
        try {
            this.read.lock();
            Map<String, Lease<InstanceInfo>> gMap = (Map)this.registry.get(registrant.getAppName());
            EurekaMonitors.REGISTER.increment(isReplication);
            if (gMap == null) {
                //map<服务名，map<实例id,实例信息>>
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

            InstanceStatus overriddenStatusFromMap = (InstanceStatus)this.overriddenInstanceStatusMap.get(registrant.getId());
            if (overriddenStatusFromMap != null) {
                logger.info("Storing overridden status {} from map", overriddenStatusFromMap);
                registrant.setOverriddenStatus(overriddenStatusFromMap);
            }

            InstanceStatus overriddenInstanceStatus = this.getOverriddenInstanceStatus(registrant, existingLease, isReplication);
            registrant.setStatusWithoutDirty(overriddenInstanceStatus);
            if (InstanceStatus.UP.equals(registrant.getStatus())) {
                lease.serviceUp();
            }

            registrant.setActionType(ActionType.ADDED);
            this.recentlyChangedQueue.add(new AbstractInstanceRegistry.RecentlyChangedItem(lease));
            registrant.setLastUpdatedTimestamp();
            //使得readWriteCacheMap无效
            this.invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
            logger.info("Registered instance {}/{} with status {} (replication={})", new Object[]{registrant.getAppName(), registrant.getId(), registrant.getStatus(), isReplication});
        } finally {
            this.read.unlock();
        }

    }
//className:AbstractInstanceRegistry
```

```java
public void invalidate(Key... keys) {
        Key[] var2 = keys;
        int var3 = keys.length;

        for(int var4 = 0; var4 < var3; ++var4) {
            Key key = var2[var4];
            logger.debug("Invalidating the response cache key : {} {} {} {}, {}", new Object[]{key.getEntityType(), key.getName(), key.getVersion(), key.getType(), key.getEurekaAccept()});
            //readWriteCacheMap缓存  点击进入 com.google.common.cache.LocalCache
            //进去之后有删除的逻辑
            this.readWriteCacheMap.invalidate(key);
            Collection<Key> keysWithRegions = this.regionSpecificKeys.get(key);
            if (null != keysWithRegions && !keysWithRegions.isEmpty()) {
                Iterator var7 = keysWithRegions.iterator();

                while(var7.hasNext()) {
                    Key keysWithRegion = (Key)var7.next();
                    logger.debug("Invalidating the response cache key : {} {} {} {} {}", new Object[]{key.getEntityType(), key.getName(), key.getVersion(), key.getType(), key.getEurekaAccept()});
                    this.readWriteCacheMap.invalidate(keysWithRegion);
                }
            }
        }

    }
```

```java
//服务获取
@GET
    public Response getApplication(@PathParam("version") String version, @HeaderParam("Accept") String acceptHeader, @HeaderParam("X-Eureka-Accept") String eurekaAccept) {
        if (!this.registry.shouldAllowAccess(false)) {
            return Response.status(Status.FORBIDDEN).build();
        } else {
            EurekaMonitors.GET_APPLICATION.increment();
            CurrentRequestVersion.set(Version.toEnum(version));
            KeyType keyType = KeyType.JSON;
            if (acceptHeader == null || !acceptHeader.contains("json")) {
                keyType = KeyType.XML;
            }

            Key cacheKey = new Key(EntityType.Application, this.appName, keyType, CurrentRequestVersion.get(), EurekaAccept.fromString(eurekaAccept));
            String payLoad = this.responseCache.get(cacheKey);
            CurrentRequestVersion.remove();
            if (payLoad != null) {
                logger.debug("Found: {}", this.appName);
                return Response.ok(payLoad).build();
            } else {
                logger.debug("Not Found: {}", this.appName);
                return Response.status(Status.NOT_FOUND).build();
            }
        }
    }
//className :ApplicationResource
```

```java
 @VisibleForTesting
    ResponseCacheImpl.Value getValue(Key key, boolean useReadOnlyCache) {
        ResponseCacheImpl.Value payload = null;

        try {
            //默认useReadOnlyCache 是true 但是可以通过use-read-only-response-cache: false 进行设置
            if (useReadOnlyCache) {
                ResponseCacheImpl.Value currentPayload = (ResponseCacheImpl.Value)this.readOnlyCacheMap.get(key);
                if (currentPayload != null) {
                    payload = currentPayload;
                } else {
                    //readWriteCacheMap 和 readOnlyCacheMap 之间的数据30s同步一次
                    payload = (ResponseCacheImpl.Value)this.readWriteCacheMap.get(key);
                    this.readOnlyCacheMap.put(key, payload);
                }
            } else {
                payload = (ResponseCacheImpl.Value)this.readWriteCacheMap.get(key);
            }
        } catch (Throwable var5) {
            logger.error("Cannot get value for key : {}", key, var5);
        }

        return payload;
    }
```

```java
//readWriteCacheMap 和 readOnlyCacheMap 之间数据同步
ResponseCacheImpl(EurekaServerConfig serverConfig, ServerCodecs serverCodecs, AbstractInstanceRegistry registry) {
        this.serverConfig = serverConfig;
        this.serverCodecs = serverCodecs;
        this.shouldUseReadOnlyResponseCache = serverConfig.shouldUseReadOnlyResponseCache();
        this.registry = registry;
        long responseCacheUpdateIntervalMs = serverConfig.getResponseCacheUpdateIntervalMs();
        this.readWriteCacheMap = CacheBuilder.newBuilder().initialCapacity(serverConfig.getInitialCapacityOfResponseCache()).expireAfterWrite(serverConfig.getResponseCacheAutoExpirationInSeconds(), TimeUnit.SECONDS).removalListener(new RemovalListener<Key, ResponseCacheImpl.Value>() {
            public void onRemoval(RemovalNotification<Key, ResponseCacheImpl.Value> notification) {
                Key removedKey = (Key)notification.getKey();
                if (removedKey.hasRegions()) {
                    Key cloneWithNoRegions = removedKey.cloneWithoutRegions();
                    ResponseCacheImpl.this.regionSpecificKeys.remove(cloneWithNoRegions, removedKey);
                }

            }
        }).build(new CacheLoader<Key, ResponseCacheImpl.Value>() {
            public ResponseCacheImpl.Value load(Key key) throws Exception {
                if (key.hasRegions()) {
                    Key cloneWithNoRegions = key.cloneWithoutRegions();
                    ResponseCacheImpl.this.regionSpecificKeys.put(cloneWithNoRegions, key);
                }

                ResponseCacheImpl.Value value = ResponseCacheImpl.this.generatePayload(key);
                return value;
            }
        });
        if (this.shouldUseReadOnlyResponseCache) {
            // 进入 这个getCacheUpdateTask
            this.timer.schedule(this.getCacheUpdateTask(), new Date(System.currentTimeMillis() / responseCacheUpdateIntervalMs * responseCacheUpdateIntervalMs + responseCacheUpdateIntervalMs), responseCacheUpdateIntervalMs);
        }

        try {
            Monitors.registerObject(this);
        } catch (Throwable var7) {
            logger.warn("Cannot register the JMX monitor for the InstanceRegistry", var7);
        }

    }
//className:ResponseCacheImpl
////////////////////////////////////////////////////////////////////////////
private TimerTask getCacheUpdateTask() {
        return new TimerTask() {
            public void run() {
                ResponseCacheImpl.logger.debug("Updating the client cache from response cache");
                Iterator var1 = ResponseCacheImpl.this.readOnlyCacheMap.keySet().iterator();

                while(var1.hasNext()) {
                    Key key = (Key)var1.next();
                    if (ResponseCacheImpl.logger.isDebugEnabled()) {
                        ResponseCacheImpl.logger.debug("Updating the client cache from response cache for key : {} {} {} {}", new Object[]{key.getEntityType(), key.getName(), key.getVersion(), key.getType()});
                    }

                    try {
                        CurrentRequestVersion.set(key.getVersion());
                       //从readWriteCacheMap取
                        ResponseCacheImpl.Value cacheValue = 
                            (ResponseCacheImpl.Value)ResponseCacheImpl.this.readWriteCacheMap.get(key); //此次的get 会调用localcache 中的getOrLoad方法
                        
                        ResponseCacheImpl.Value currentCacheValue = (ResponseCacheImpl.Value)ResponseCacheImpl.this.readOnlyCacheMap.get(key);
                        if (cacheValue != currentCacheValue) {
                            //放入readOnlyCacheMap
                            ResponseCacheImpl.this.readOnlyCacheMap.put(key, cacheValue);
                        }
                    } catch (Throwable var8) {
                        ResponseCacheImpl.logger.error("Error while updating the client cache from response cache for key {}", key.toStringCompact(), var8);
                    } finally {
                        CurrentRequestVersion.remove();
                    }
                }

            }
        };
    }
```

#### 三级缓存之间的流程

register(一级)：当有服务注册时会给注册表进行注入，同时让二级缓存（ readWriteCacheMap )失效。

readWriteCacheMap （二级）：默认每30s 二级缓存和三级缓存之间会进行数据同步。

 readOnly （三级缓存）：use-read-only-response-cache 这个设置的默认值时true，当为true时会先在三级缓存中获取服务，查询不到时会去二级缓存中查，然后放入3级缓存，当二级缓存失效时， 触发guava的回调函数从注册表中同步 （上面源码 localcache 中的getOrLoad方法）

#### 三级缓存的好处

进行的读写分离，写的时候往一级缓存写，读的时候从二级或三级缓存读。

### 为什么说eureka实现了CAP 中AP 没有实现C

因为3级缓存的存在 是的读写时分离的。