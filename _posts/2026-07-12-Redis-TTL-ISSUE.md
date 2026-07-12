---
title: "[트러블슈팅] CacheManager 래핑 시 afterPropertiesSet() 미호출로 인한 TTL 무시 이슈"
date: 2026-07-12 00:00:00 +0900
categories: [트러블슈팅]
tags: [Spring, RedisCacheManager, CacheManager, InitializingBean, "@Cacheable", TTL, 트러블슈팅]
---

## 1. 문제 상황

```java
@Cacheable(cacheNames = "session", key = "'session:' + #sessionKey")
public SessionDto getSession(String sessionKey) {
    // ..
} 
```

현재 프로젝트에서 스프링에서 제공하는 `@Cacheable`, `CacheManager`를 통해 간단한 설정만으로 Redis 캐싱을 적용하여 사용 중이었다.

`CacheManager` 설정을 통해 `cacheName` 별로 TTL 설정하였고, 캐시 hit/miss를 커스텀하게 로깅을 남기기 위해 `CacheManager`를 구현한 `LoggingCacheManager` 추가하여 사용 중이었다.

```java
public class LoggingCacheManager implements CacheManager {

    private final CacheManager delegate;

    public LoggingCacheManager(CacheManager delegate) {
        this.delegate = delegate;
    }

    @Override
    public Cache getCache(String name) {
        Cache cache = delegate.getCache(name);
        if (cache != null) {
            log.info("[Cache HIT] cacheName={}", name);
        } else {
            log.info("[Cache MISS] cacheName={}", name);
        }
        return cache;
    }

    @Override
    public Collection<String> getCacheNames() {
        return delegate.getCacheNames();
    }
}

@Configuration
public class CacheConfig {

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory redisConnectionFactory) {
        // 1. 기본 설정
        RedisCacheConfiguration defaultCacheConfig = RedisCacheConfiguration.defaultCacheConfig()
            // ..
            .entryTtl(Duration.ofMinutes(10)); // 기본 TTL: 10분

        // 2. 캐시 이름(Value)별로 상이한 TTL 설정
        Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();
        cacheConfigurations.put("session", defaultCacheConfig.entryTtl(Duration.ofSeconds(30)));
        // ..

        // 3. RedisCacheManager 빌드 및 반환
        RedisCacheManager redisCacheManager = RedisCacheManager.builder(redisConnectionFactory)
                                        .cacheDefaults(defaultCacheConfig) // 기본 설정 적용
                                        .withInitialCacheConfigurations(cacheConfigurations) // 캐시별 특수 설정 적용
                                        // ..
                                        .build();
        return new LoggingCacheManager(redisCacheManager);
    }
}
```

하지만, 테스트 과정에서 cacheName = session인 TTL을 30초로 기대했는데, 기본값인 10분으로 TTL이 적용되는 이슈가 있었다.

```
127.0.0.1:6379> TTL session:123
(integer) 600  # 기대: 30, 실제: 600 (10분)
```

QA 중에 발견하게 되어 정말 다행이었다..


## 2. 원인

`LoggingCacheManager` 래핑으로 인한 `afterPropertiesSet()` 미호출로 인해서 발생한 이슈였다.

1. `withInitialCacheConfigurations` — `initialCacheConfiguration` 필드에 저장만 됨
  - `withInitialCacheConfigurations()`는 빌더의 `initialCaches`에 저장되고, `build()` 시 `RedisCacheManager` 생성자로 전달되어 `initialCacheConfiguration` 필드에 담긴다. 
  - 이 시점엔 `cacheMap`에 아무것도 등록되지 않는다.

```java
// RedisCacheManagerBuilder.withInitialCacheConfigurations()
public RedisCacheManagerBuilder withInitialCacheConfigurations(Map<String, RedisCacheConfiguration> cacheConfigurations) {
    this.initialCaches.putAll(cacheConfigurations); // 빌더의 initialCaches에 저장
    return this;
}

// RedisCacheManagerBuilder.build() → newRedisCacheManager()
private RedisCacheManager newRedisCacheManager(RedisCacheWriter cacheWriter) {
    // initialCaches를 RedisCacheManager 생성자에 전달
    // 생성자 내부에서 this.initialCacheConfiguration.putAll(initialCacheConfigurations)
    return new RedisCacheManager(cacheWriter, this.cacheDefaults(), this.allowRuntimeCacheCreation, this.initialCaches);
}
```

2. `afterPropertiesSet()` → `loadCaches()` → `cacheMap` 초기화
  - `RedisCacheManager`는 `AbstractCacheManager`를 상속하며 `InitializingBean`을 구현한다. 
  - Spring은 빈 등록 시 `afterPropertiesSet()`을 자동 호출하고, 이 흐름을 통해 `initialCacheConfiguration`을 읽어 cacheName별 TTL이 적용된 `RedisCache`가 `cacheMap`에 등록된다.

```java
public class RedisCacheManager extends AbstractTransactionSupportingCacheManager { }

public abstract class AbstractTransactionSupportingCacheManager extends AbstractCacheManager { }

public abstract class AbstractCacheManager implements CacheManager, InitializingBean {

    public void afterPropertiesSet() {
        this.initializeCaches();
    }

    public void initializeCaches() {
        Collection<? extends Cache> caches = this.loadCaches(); // RedisCacheManager.loadCaches() 호출
        synchronized(this.cacheMap) {
            this.cacheMap.clear();
            for (Cache cache : caches) {
                this.cacheMap.put(cache.getName(), this.decorateCache(cache)); // cacheMap에 등록
            }
        }
    }
}

// RedisCacheManager.loadCaches() — initialCacheConfiguration을 읽어 캐시 생성
protected Collection<RedisCache> loadCaches() {
    return this.getInitialCacheConfiguration().entrySet().stream()
        .map((entry) -> this.createRedisCache(entry.getKey(), entry.getValue())) // TTL 30초 적용
        .toList();
}
```

**③ `afterPropertiesSet()` 미호출 시 — `getMissingCache()` fallback**

`builder.build()` 결과를 `LoggingCacheManager`로 감싸 반환하면 Spring 빈은 `LoggingCacheManager`이고, 내부 `RedisCacheManager(delegate)`는 일반 필드로 존재하게 되어 `delegate.afterPropertiesSet()`이 호출되지 않는다. 결과적으로 `cacheMap`은 비어있는 상태가 된다.

이 상태에서 `getCache("session")`이 호출되면 `cacheMap.get("session")`이 null을 반환하고 `getMissingCache()`로 넘어간다. 여기서 `initialCacheConfiguration`은 완전히 무시되고 `defaultCacheConfiguration`(10분)으로 캐시를 새로 생성하는 것이 실제 원인이었다.

```java
// AbstractCacheManager.getCache()
public Cache getCache(String name) {
    Cache cache = this.cacheMap.get(name); // cacheMap이 비어있으므로 null
    if (cache != null) {
        return cache;
    }
    Cache missingCache = this.getMissingCache(name); // fallback
    // ...
}

// RedisCacheManager.getMissingCache()
protected RedisCache getMissingCache(String name) {
    return this.isAllowRuntimeCacheCreation()
        ? this.createRedisCache(name, this.getDefaultCacheConfiguration()) // initialCacheConfiguration 무시, defaultCacheConfiguration(10분)으로 생성
        : null;
}
```

## 3. 해결 방법

`LoggingCacheManager`가 `InitializingBean`을 함께 구현하고, `afterPropertiesSet()`을 내부 `RedisCacheManager`에 위임하도록 수정한다.

```java
public class LoggingCacheManager implements CacheManager, InitializingBean {

    private final CacheManager delegate;

    public LoggingCacheManager(CacheManager delegate) {
        this.delegate = delegate;
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        if (delegate instanceof InitializingBean initializingBean) {
            initializingBean.afterPropertiesSet(); // cacheMap 초기화 위임
        }
    }

    @Override
    public Cache getCache(String name) {
        Cache cache = delegate.getCache(name);
        if (cache != null) {
            log.info("[Cache HIT] cacheName={}", name);
        } else {
            log.info("[Cache MISS] cacheName={}", name);
        }
        return cache;
    }

    @Override
    public Collection<String> getCacheNames() {
        return delegate.getCacheNames();
    }
}
```

Spring이 `LoggingCacheManager` 빈을 초기화할 때 `afterPropertiesSet()`을 호출하고, 이를 내부 `CacheManager`에 위임함으로써 `cacheMap`이 정상적으로 초기화되어 캐시 이름별 TTL 설정이 올바르게 적용된다.
