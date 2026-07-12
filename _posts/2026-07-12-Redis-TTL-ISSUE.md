---
title: "[트러블슈팅] CacheManager 래핑 시 afterPropertiesSet() 미호출로 인한 TTL 무시 이슈"
date: 2026-07-12 23:27:00 +0900
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

`RedisCacheManager`는 `AbstractCacheManager`를 상속하며 `InitializingBean`을 구현한다. Spring은 빈 등록 시 `afterPropertiesSet()`을 자동 호출해 `cacheMap`을 초기화하는데, `builder.build()` 결과를 `LoggingCacheManager`로 감싸 반환하면 Spring 빈은 `LoggingCacheManager`이고, 내부 `RedisCacheManager`는 일반 Java 객체로 존재하게 되어 `afterPropertiesSet()`이 호출되지 않는다.

```java
public class LoggingCacheManager implements CacheManager { }

public class RedisCacheManager extends AbstractTransactionSupportingCacheManager { }

public abstract class AbstractTransactionSupportingCacheManager extends AbstractCacheManager {}

public abstract class AbstractCacheManager implements CacheManager, InitializingBean {

    // Spring 빈 등록 시 afterPropertiesSet()을 자동 호출해 cacheMap을 초기화
    public void afterPropertiesSet() {
        this.initializeCaches();
    }

    public void initializeCaches() {
        Collection<? extends Cache> caches = this.loadCaches();
        synchronized(this.cacheMap) {
            this.cacheNames = Collections.emptySet();
            this.cacheMap.clear();
            Set<String> cacheNames = new LinkedHashSet(caches.size());

            for(Cache cache : caches) {
                String name = cache.getName();
                this.cacheMap.put(name, this.decorateCache(cache));
                cacheNames.add(name);
            }

            this.cacheNames = Collections.unmodifiableSet(cacheNames);
        }
    }
    
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
