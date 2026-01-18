---
layout: post
title: "Redis 캐시 스탬피드(Cache Stampede) 완전 정리: 동시성 캐시 미스 방지 전략"
date: 2026-01-18
categories: [redis, architecture, spring, performance]
tags: [Redis, CacheStampede, 캐시스탬피드, 캐시스톰프, 동시성, 분산락, 성능최적화, SpringCache, 캐시관리]
---

이전 글에서 Kafka DLQ를 통해 실패한 메시지를 효과적으로 관리하는 방법을 다뤘습니다. 분산 시스템에서는 메시지 처리뿐만 아니라 **캐시 관리**에서도 동시성 문제가 발생할 수 있습니다. 특히 **캐시 스탬피드(Cache Stampede)**는 동시에 많은 요청이 캐시 미스를 발생시켜 시스템에 큰 부하를 주는 대표적인 문제입니다.

이번 글에서는 **캐시 스탬피드 문제**를 분석하고, **Redis를 활용한 해결 전략**을 정리해보겠습니다.

---

## 1. 캐시 스탬피드(Cache Stampede)란?

### 1.1 문제 정의

**캐시 스탬피드(Cache Stampede, 캐시 스톰프):**

- **동시에 많은 요청이 캐시 미스(Cache Miss)를 발생**시킬 때
- **모든 요청이 동시에 DB나 외부 시스템에서 데이터를 조회**하려고 시도
- 결과적으로 **DB나 외부 시스템에 과부하** 발생
- 캐시가 만료되거나 처음 로드될 때 특히 심각

**비유:**

```
🚶‍♂️🚶‍♂️🚶‍♂️🚶‍♂️🚶‍♂️ (수많은 사람들)
         ↓
      🚪 (캐시 만료)
         ↓
      💥 (모두 동시에 DB 조회)
         ↓
      🔥 (DB 과부하로 시스템 다운)
```

### 1.2 발생 시나리오

**시나리오 1: 캐시 만료 시 동시 요청**

```
시간: 10:00:00 (캐시 만료 시점)

요청 1: GET /api/products/123
  → Redis에 데이터 없음 (캐시 미스)
  → DB 조회 시작
  
요청 2: GET /api/products/123 (거의 동시에)
  → Redis에 데이터 없음 (캐시 미스)
  → DB 조회 시작
  
요청 3: GET /api/products/123 (거의 동시에)
  → Redis에 데이터 없음 (캐시 미스)
  → DB 조회 시작
  
... (수백, 수천 개의 동시 요청)
  
→ 결과: DB에 동일한 쿼리가 수백 번 실행됨
→ DB 과부하, 응답 시간 증가, 시스템 다운 가능
```

**시나리오 2: 인기 상품 캐시 미스**

```java
// 인기 상품 정보를 캐시에서 조회
@Cacheable(value = "popular-products", key = "#productId")
public Product getPopularProduct(Long productId) {
    // 캐시에 없으면 DB 조회
    return productRepository.findById(productId)
        .orElseThrow(() -> new ProductNotFoundException());
}
```

```
캐시 만료 → 수천 명의 사용자가 동시에 인기 상품 조회
→ 모두 캐시 미스 발생
→ 모두 DB에서 동일한 상품 정보 조회
→ DB 과부하
```

---

## 2. 캐시 스탬피드의 영향

### 2.1 성능 저하

**정상적인 캐시 동작:**

```
요청 1: Redis 조회 → 캐시 히트 → 즉시 반환 (1ms)
요청 2: Redis 조회 → 캐시 히트 → 즉시 반환 (1ms)
요청 3: Redis 조회 → 캐시 히트 → 즉시 반환 (1ms)
```

**캐시 스탬피드 발생 시:**

```
요청 1-100: Redis 조회 → 캐시 미스 → DB 조회 (각각 100ms)
요청 101-200: Redis 조회 → 캐시 미스 → DB 조회 대기 (200ms+)
요청 201-300: Redis 조회 → 캐시 미스 → DB 조회 대기 (300ms+)
...
```

### 2.2 시스템 안정성 문제

- **DB 커넥션 풀 고갈**: 동시에 너무 많은 쿼리 실행
- **DB CPU/메모리 부족**: 과도한 부하로 인한 성능 저하
- **응답 시간 증가**: 대기 시간 증가로 인한 타임아웃
- **서비스 장애**: 연쇄적인 장애로 이어질 수 있음

### 2.3 실제 발생 예시

**전자상거래 이벤트:**

```
이벤트 시작: 00:00:00
인기 상품 캐시 만료: 00:00:00
동시 접속자: 10,000명

→ 10,000명이 모두 동일한 상품 정보를 DB에서 조회
→ DB 응답 시간: 평균 10ms → 5초 이상으로 증가
→ 서비스 장애 발생
```

---

## 3. 해결 전략 1: Lock 기반 해결 (분산 락)

### 3.1 기본 개념

**Lock을 사용한 캐시 스탬피드 방지:**

- 첫 번째 요청만 DB에서 데이터 조회
- 나머지 요청은 Lock을 획득하지 못하면 **대기**
- 첫 번째 요청이 캐시에 데이터를 저장하면, 나머지 요청은 캐시에서 조회

**동작 흐름:**

```
요청 1: Lock 획득 성공 → DB 조회 → Redis에 저장 → Lock 해제
요청 2: Lock 획득 실패 → 대기 → Redis에서 조회
요청 3: Lock 획득 실패 → 대기 → Redis에서 조회
```

### 3.2 Redis 분산 락을 활용한 구현

**Spring Boot + Redis (Redisson) 예시:**

```java
@Service
@Slf4j
public class ProductService {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private RedissonClient redissonClient;  // Redisson 클라이언트
    
    private static final String CACHE_KEY_PREFIX = "product:";
    private static final String LOCK_KEY_PREFIX = "lock:product:";
    private static final int CACHE_TTL_SECONDS = 3600;  // 1시간
    
    public Product getProduct(Long productId) {
        String cacheKey = CACHE_KEY_PREFIX + productId;
        
        // 1. 캐시에서 조회
        Product cachedProduct = (Product) redisTemplate.opsForValue().get(cacheKey);
        if (cachedProduct != null) {
            log.debug("Cache hit: productId={}", productId);
            return cachedProduct;
        }
        
        // 2. 캐시 미스 → Lock 획득 시도
        String lockKey = LOCK_KEY_PREFIX + productId;
        RLock lock = redissonClient.getLock(lockKey);
        
        try {
            // Lock 획득 시도 (최대 3초 대기, 10초 후 자동 해제)
            boolean lockAcquired = lock.tryLock(3, 10, TimeUnit.SECONDS);
            
            if (lockAcquired) {
                try {
                    log.info("Lock acquired: productId={}", productId);
                    
                    // 3. Double-check: Lock을 획득한 후 다시 캐시 확인
                    cachedProduct = (Product) redisTemplate.opsForValue().get(cacheKey);
                    if (cachedProduct != null) {
                        log.debug("Cache populated by another thread: productId={}", productId);
                        return cachedProduct;
                    }
                    
                    // 4. DB에서 조회
                    log.info("Loading from DB: productId={}", productId);
                    Product product = productRepository.findById(productId)
                        .orElseThrow(() -> new ProductNotFoundException(productId));
                    
                    // 5. Redis에 저장
                    redisTemplate.opsForValue().set(
                        cacheKey, 
                        product, 
                        CACHE_TTL_SECONDS, 
                        TimeUnit.SECONDS
                    );
                    
                    log.info("Cache updated: productId={}", productId);
                    return product;
                    
                } finally {
                    lock.unlock();
                }
            } else {
                // Lock 획득 실패 → 잠시 대기 후 캐시 재조회
                log.debug("Lock acquisition failed, retrying cache: productId={}", productId);
                Thread.sleep(100);  // 100ms 대기
                
                cachedProduct = (Product) redisTemplate.opsForValue().get(cacheKey);
                if (cachedProduct != null) {
                    return cachedProduct;
                }
                
                // 캐시가 아직 없으면 DB 조회 (최후의 수단)
                log.warn("Cache still not available after lock wait: productId={}", productId);
                return productRepository.findById(productId)
                    .orElseThrow(() -> new ProductNotFoundException(productId));
            }
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            log.error("Lock acquisition interrupted: productId={}", productId, e);
            throw new RuntimeException("Failed to acquire lock", e);
        }
    }
}
```

### 3.3 Spring Cache + AOP를 활용한 구현

**AOP를 사용한 자동화된 Lock 처리:**

```java
@Component
@Aspect
@Slf4j
public class CacheStampedeAspect {
    
    @Autowired
    private RedissonClient redissonClient;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Around("@annotation(cacheable) && execution(* *(..))")
    public Object preventCacheStampede(
            ProceedingJoinPoint joinPoint, 
            Cacheable cacheable) throws Throwable {
        
        // 1. 캐시 키 생성
        String cacheKey = generateCacheKey(joinPoint, cacheable);
        String lockKey = "lock:" + cacheKey;
        
        // 2. 캐시에서 조회
        Object cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return cached;
        }
        
        // 3. Lock 획득 시도
        RLock lock = redissonClient.getLock(lockKey);
        
        try {
            if (lock.tryLock(3, 10, TimeUnit.SECONDS)) {
                try {
                    // Double-check
                    cached = redisTemplate.opsForValue().get(cacheKey);
                    if (cached != null) {
                        return cached;
                    }
                    
                    // 원본 메서드 실행 (DB 조회)
                    Object result = joinPoint.proceed();
                    
                    // 캐시에 저장
                    if (result != null) {
                        long ttl = cacheable.expiration() > 0 
                            ? cacheable.expiration() 
                            : 3600;  // 기본 1시간
                        redisTemplate.opsForValue().set(
                            cacheKey, 
                            result, 
                            ttl, 
                            TimeUnit.SECONDS
                        );
                    }
                    
                    return result;
                } finally {
                    lock.unlock();
                }
            } else {
                // Lock 획득 실패 → 재시도
                Thread.sleep(100);
                cached = redisTemplate.opsForValue().get(cacheKey);
                if (cached != null) {
                    return cached;
                }
                // 최후의 수단: 원본 메서드 실행
                return joinPoint.proceed();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("Lock acquisition interrupted", e);
        }
    }
    
    private String generateCacheKey(ProceedingJoinPoint joinPoint, Cacheable cacheable) {
        // 캐시 키 생성 로직 (SpEL 표현식 처리)
        // 간단한 예시
        Object[] args = joinPoint.getArgs();
        return cacheable.value()[0] + ":" + Arrays.toString(args);
    }
}
```

---

## 4. 해결 전략 2: Probabilistic Early Expiration (확률적 조기 만료)

### 4.1 기본 개념

**확률적 조기 만료 전략:**

- 캐시 TTL을 **랜덤하게 조정**하여 동시 만료 방지
- 예: TTL을 `3600 ± 10%`로 설정 (3240~3960초)
- **가장 빠르게 만료되는 캐시가 먼저 갱신**되어 다른 캐시의 만료를 방지

**동작 원리:**

```
캐시 A: TTL = 3240초 (빠르게 만료)
캐시 B: TTL = 3600초 (정상 만료)
캐시 C: TTL = 3960초 (늦게 만료)

→ 캐시 A가 먼저 만료 → 백그라운드에서 갱신
→ 캐시 B, C는 아직 유효 → 캐시 A 갱신 후 함께 사용
→ 동시 만료 방지
```

### 4.2 구현 예시

```java
@Service
@Slf4j
public class ProductServiceWithProbabilisticExpiration {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    private static final int BASE_TTL_SECONDS = 3600;  // 기본 1시간
    private static final double JITTER_RATE = 0.1;  // ±10% 변동
    
    public Product getProduct(Long productId) {
        String cacheKey = "product:" + productId;
        
        // 1. 캐시에서 조회
        Product cachedProduct = (Product) redisTemplate.opsForValue().get(cacheKey);
        if (cachedProduct != null) {
            log.debug("Cache hit: productId={}", productId);
            return cachedProduct;
        }
        
        // 2. DB에서 조회
        log.info("Loading from DB: productId={}", productId);
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        
        // 3. 확률적 TTL로 캐시에 저장
        int ttl = calculateProbabilisticTTL();
        redisTemplate.opsForValue().set(
            cacheKey, 
            product, 
            ttl, 
            TimeUnit.SECONDS
        );
        
        log.info("Cache updated with TTL {} seconds: productId={}", ttl, productId);
        return product;
    }
    
    /**
     * 확률적 TTL 계산
     * BASE_TTL ± (BASE_TTL * JITTER_RATE * random)
     */
    private int calculateProbabilisticTTL() {
        Random random = new Random();
        double jitter = (random.nextDouble() * 2 - 1) * JITTER_RATE;  // -0.1 ~ +0.1
        int ttl = (int) (BASE_TTL_SECONDS * (1 + jitter));
        
        // 최소값 보장 (예: 최소 50%는 유지)
        return Math.max(ttl, BASE_TTL_SECONDS / 2);
    }
}
```

---

## 5. 해결 전략 3: Background Refresh (백그라운드 갱신)

### 5.1 기본 개념

**백그라운드 갱신 전략:**

- 캐시가 만료되기 **전에 백그라운드에서 미리 갱신**
- 사용자는 만료된 캐시를 즉시 받지만, 백그라운드에서 새로운 데이터로 갱신
- 캐시 만료 시점에 DB 부하 없음

**동작 흐름:**

```
TTL: 3600초 (1시간)
갱신 시점: 3000초 후 (만료 10분 전)

T=0초: 캐시 생성
T=3000초: 백그라운드 갱신 시작 (사용자는 기존 캐시 사용)
T=3100초: 백그라운드 갱신 완료
T=3600초: 캐시 만료 → 이미 새로운 데이터로 갱신되어 있음
```

### 5.2 Spring Cache + Scheduled Task 구현

```java
@Service
@Slf4j
public class ProductServiceWithBackgroundRefresh {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private RedissonClient redissonClient;
    
    private static final String CACHE_KEY_PREFIX = "product:";
    private static final String REFRESH_FLAG_PREFIX = "refresh:product:";
    private static final int CACHE_TTL_SECONDS = 3600;
    private static final int REFRESH_BEFORE_EXPIRY_SECONDS = 600;  // 만료 10분 전
    
    /**
     * 정기적으로 인기 상품 캐시를 갱신
     */
    @Scheduled(fixedRate = 300000)  // 5분마다 실행
    public void refreshPopularProducts() {
        log.info("Starting background cache refresh");
        
        // 인기 상품 ID 목록 조회
        List<Long> popularProductIds = productRepository.findPopularProductIds();
        
        for (Long productId : popularProductIds) {
            refreshCacheInBackground(productId);
        }
        
        log.info("Background cache refresh completed: {} products", popularProductIds.size());
    }
    
    /**
     * 백그라운드에서 캐시 갱신
     */
    @Async
    public void refreshCacheInBackground(Long productId) {
        String cacheKey = CACHE_KEY_PREFIX + productId;
        String refreshFlagKey = REFRESH_FLAG_PREFIX + productId;
        
        // 1. 이미 갱신 중인지 확인
        Boolean isRefreshing = redisTemplate.hasKey(refreshFlagKey);
        if (Boolean.TRUE.equals(isRefreshing)) {
            log.debug("Cache refresh already in progress: productId={}", productId);
            return;
        }
        
        // 2. 캐시 TTL 확인
        Long ttl = redisTemplate.getExpire(cacheKey);
        if (ttl == null || ttl > REFRESH_BEFORE_EXPIRY_SECONDS) {
            log.debug("Cache still fresh: productId={}, ttl={}", productId, ttl);
            return;
        }
        
        // 3. 갱신 플래그 설정
        String lockKey = "lock:refresh:" + productId;
        RLock lock = redissonClient.getLock(lockKey);
        
        try {
            if (lock.tryLock(1, 5, TimeUnit.SECONDS)) {
                try {
                    // Double-check
                    ttl = redisTemplate.getExpire(cacheKey);
                    if (ttl == null || ttl > REFRESH_BEFORE_EXPIRY_SECONDS) {
                        return;
                    }
                    
                    // 갱신 플래그 설정
                    redisTemplate.opsForValue().set(
                        refreshFlagKey, 
                        "refreshing", 
                        60, 
                        TimeUnit.SECONDS
                    );
                    
                    // DB에서 조회
                    log.info("Background refresh: productId={}", productId);
                    Product product = productRepository.findById(productId)
                        .orElse(null);
                    
                    if (product != null) {
                        // 캐시 업데이트
                        redisTemplate.opsForValue().set(
                            cacheKey, 
                            product, 
                            CACHE_TTL_SECONDS, 
                            TimeUnit.SECONDS
                        );
                        log.info("Cache refreshed: productId={}", productId);
                    }
                    
                } finally {
                    // 갱신 플래그 제거
                    redisTemplate.delete(refreshFlagKey);
                    lock.unlock();
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            log.error("Background refresh interrupted: productId={}", productId, e);
        }
    }
    
    /**
     * 일반 조회 메서드
     */
    public Product getProduct(Long productId) {
        String cacheKey = CACHE_KEY_PREFIX + productId;
        
        // 캐시에서 조회
        Product cachedProduct = (Product) redisTemplate.opsForValue().get(cacheKey);
        if (cachedProduct != null) {
            // 캐시가 곧 만료될 것 같으면 백그라운드 갱신 트리거
            Long ttl = redisTemplate.getExpire(cacheKey);
            if (ttl != null && ttl < REFRESH_BEFORE_EXPIRY_SECONDS) {
                refreshCacheInBackground(productId);
            }
            
            return cachedProduct;
        }
        
        // 캐시 미스 → DB 조회
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        
        // 캐시에 저장
        redisTemplate.opsForValue().set(
            cacheKey, 
            product, 
            CACHE_TTL_SECONDS, 
            TimeUnit.SECONDS
        );
        
        return product;
    }
}
```

---

## 6. 해결 전략 4: Cache Warming (캐시 워밍업)

### 6.1 기본 개념

**캐시 워밍업 전략:**

- **서비스 시작 시 또는 정기적으로 인기 데이터를 미리 캐시에 로드**
- 캐시 만료 전에 미리 갱신하여 스탬피드 방지
- 예측 가능한 트래픽 패턴에 효과적

### 6.2 구현 예시

```java
@Component
@Slf4j
public class CacheWarmingService {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private ProductService productService;
    
    /**
     * 애플리케이션 시작 시 캐시 워밍업
     */
    @EventListener(ApplicationReadyEvent.class)
    public void warmupCache() {
        log.info("Starting cache warmup...");
        
        // 인기 상품 목록 조회
        List<Long> popularProductIds = productRepository.findPopularProductIds(100);
        
        // 병렬로 캐시에 로드
        popularProductIds.parallelStream()
            .forEach(productId -> {
                try {
                    productService.getProduct(productId);
                    log.debug("Cache warmed up: productId={}", productId);
                } catch (Exception e) {
                    log.error("Failed to warm up cache: productId={}", productId, e);
                }
            });
        
        log.info("Cache warmup completed: {} products", popularProductIds.size());
    }
    
    /**
     * 정기적으로 캐시 워밍업
     */
    @Scheduled(cron = "0 0 */6 * * *")  // 6시간마다
    public void scheduledCacheWarmup() {
        log.info("Starting scheduled cache warmup...");
        warmupCache();
    }
}
```

---

## 7. 통합 전략: 여러 방법 조합

### 7.1 계층적 캐시 스탬피드 방지

**권장 조합:**

1. **Lock 기반**: 즉시 해결, 모든 상황에 적용 가능
2. **Probabilistic Early Expiration**: 추가 보호, 구현 간단
3. **Background Refresh**: 예측 가능한 트래픽에 효과적
4. **Cache Warming**: 서비스 시작 시 필수

### 7.2 통합 구현 예시

```java
@Service
@Slf4j
public class ProductServiceWithStampedeProtection {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private RedissonClient redissonClient;
    
    private static final String CACHE_KEY_PREFIX = "product:";
    private static final String LOCK_KEY_PREFIX = "lock:product:";
    private static final int BASE_TTL_SECONDS = 3600;
    private static final double JITTER_RATE = 0.1;
    private static final int REFRESH_BEFORE_EXPIRY_SECONDS = 600;
    
    public Product getProduct(Long productId) {
        String cacheKey = CACHE_KEY_PREFIX + productId;
        
        // 1. 캐시에서 조회
        Product cachedProduct = (Product) redisTemplate.opsForValue().get(cacheKey);
        if (cachedProduct != null) {
            // 캐시가 곧 만료될 것 같으면 백그라운드 갱신 트리거
            Long ttl = redisTemplate.getExpire(cacheKey);
            if (ttl != null && ttl < REFRESH_BEFORE_EXPIRY_SECONDS) {
                refreshCacheInBackground(productId);
            }
            
            return cachedProduct;
        }
        
        // 2. Lock 기반 캐시 스탬피드 방지
        return getProductWithLock(productId, cacheKey);
    }
    
    private Product getProductWithLock(Long productId, String cacheKey) {
        String lockKey = LOCK_KEY_PREFIX + productId;
        RLock lock = redissonClient.getLock(lockKey);
        
        try {
            if (lock.tryLock(3, 10, TimeUnit.SECONDS)) {
                try {
                    // Double-check
                    Product cachedProduct = (Product) redisTemplate.opsForValue().get(cacheKey);
                    if (cachedProduct != null) {
                        return cachedProduct;
                    }
                    
                    // DB에서 조회
                    Product product = productRepository.findById(productId)
                        .orElseThrow(() -> new ProductNotFoundException(productId));
                    
                    // 확률적 TTL로 캐시에 저장
                    int ttl = calculateProbabilisticTTL();
                    redisTemplate.opsForValue().set(
                        cacheKey, 
                        product, 
                        ttl, 
                        TimeUnit.SECONDS
                    );
                    
                    return product;
                    
                } finally {
                    lock.unlock();
                }
            } else {
                // Lock 획득 실패 → 재시도
                Thread.sleep(100);
                Product cachedProduct = (Product) redisTemplate.opsForValue().get(cacheKey);
                if (cachedProduct != null) {
                    return cachedProduct;
                }
                
                // 최후의 수단: DB 조회
                return productRepository.findById(productId)
                    .orElseThrow(() -> new ProductNotFoundException(productId));
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("Failed to acquire lock", e);
        }
    }
    
    @Async
    private void refreshCacheInBackground(Long productId) {
        // 백그라운드 갱신 로직 (이전 예시와 동일)
        // ...
    }
    
    private int calculateProbabilisticTTL() {
        Random random = new Random();
        double jitter = (random.nextDouble() * 2 - 1) * JITTER_RATE;
        int ttl = (int) (BASE_TTL_SECONDS * (1 + jitter));
        return Math.max(ttl, BASE_TTL_SECONDS / 2);
    }
}
```

---

## 8. Spring Cache Abstraction 활용

### 8.1 CacheManager 커스터마이징

**Redis CacheManager 설정:**

```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1))
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()))
            .disableCachingNullValues();
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(defaultConfig)
            .withCacheConfiguration("products", 
                defaultConfig.entryTtl(Duration.ofHours(1)))
            .withCacheConfiguration("popular-products", 
                defaultConfig.entryTtl(Duration.ofHours(2)))
            .transactionAware()
            .build();
    }
}
```

### 8.2 Custom CacheResolver를 통한 스탬피드 방지

```java
@Component
public class StampedePreventingCacheResolver implements CacheResolver {
    
    @Autowired
    private CacheManager cacheManager;
    
    @Autowired
    private RedissonClient redissonClient;
    
    @Override
    public Collection<? extends Cache> resolveCaches(CacheOperationInvocationContext<?> context) {
        String cacheName = context.getOperation().getCacheNames().iterator().next();
        Cache cache = cacheManager.getCache(cacheName);
        
        if (cache instanceof RedisCache) {
            return Collections.singleton(new StampedePreventingCache((RedisCache) cache, redissonClient));
        }
        
        return Collections.singleton(cache);
    }
}
```

---

## 9. 모니터링 및 메트릭

### 9.1 캐시 스탬피드 감지

**주요 메트릭:**

- **캐시 히트율 (Cache Hit Rate)**: 목표 95% 이상
- **캐시 미스율 급증**: 스탬피드 의심
- **동시 DB 조회 수**: Lock 효과 측정
- **평균 응답 시간**: 캐시 히트/미스별

### 9.2 Spring Actuator + Micrometer 활용

```java
@Configuration
public class MetricsConfig {
    
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCustomizer() {
        return registry -> {
            registry.config().commonTags("application", "product-service");
        };
    }
}
```

```java
@Service
@Slf4j
public class ProductServiceWithMetrics {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    private final Counter cacheHitCounter;
    private final Counter cacheMissCounter;
    private final Counter dbQueryCounter;
    private final Timer cacheOperationTimer;
    
    public ProductServiceWithMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.cacheHitCounter = Counter.builder("cache.hits")
            .description("Cache hit count")
            .tag("cache", "product")
            .register(meterRegistry);
        
        this.cacheMissCounter = Counter.builder("cache.misses")
            .description("Cache miss count")
            .tag("cache", "product")
            .register(meterRegistry);
        
        this.dbQueryCounter = Counter.builder("db.queries")
            .description("DB query count")
            .tag("table", "product")
            .register(meterRegistry);
        
        this.cacheOperationTimer = Timer.builder("cache.operation.duration")
            .description("Cache operation duration")
            .tag("operation", "get")
            .register(meterRegistry);
    }
    
    public Product getProduct(Long productId) {
        return cacheOperationTimer.recordCallable(() -> {
            String cacheKey = "product:" + productId;
            
            Product cachedProduct = getFromCache(cacheKey);
            if (cachedProduct != null) {
                cacheHitCounter.increment();
                return cachedProduct;
            }
            
            cacheMissCounter.increment();
            
            // Lock 기반 조회
            Product product = getProductWithLock(productId, cacheKey);
            dbQueryCounter.increment();
            
            return product;
        });
    }
    
    // ... 나머지 구현
}
```

---

## 10. Best Practices

### 10.1 설계 원칙

**1. Lock 타임아웃 설정**
```java
// 너무 짧으면: Lock 획득 실패 빈번
// 너무 길면: Deadlock 위험
lock.tryLock(3, 10, TimeUnit.SECONDS);  // 3초 대기, 10초 후 자동 해제
```

**2. Double-Check 패턴 필수**
```java
// Lock 획득 후 반드시 캐시 재확인
// 다른 스레드가 이미 캐시를 채웠을 수 있음
```

**3. 확률적 TTL 범위**
```java
// 너무 작으면: 캐시 효율 감소
// 너무 크면: 스탬피드 방지 효과 감소
double jitterRate = 0.1;  // ±10% 권장
```

**4. 백그라운드 갱신 타이밍**
```java
// 만료 시점의 10~20% 전에 갱신
int refreshBeforeExpiry = (int) (ttl * 0.1);  // 10% 전
```

### 10.2 환경별 권장 사항

**개발 환경:**
- Lock 기반만으로 충분
- 모니터링 최소화

**프로덕션 환경:**
- Lock + Probabilistic TTL + Background Refresh 조합
- 모니터링 및 알림 필수
- Cache Warming 활용

**고트래픽 환경:**
- 모든 전략 통합 사용
- 다단계 캐시 (Local Cache + Redis)
- Read Replica 활용

---

## 11. 주의사항 및 제한사항

### 11.1 Lock 사용 시 주의사항

**Deadlock 위험:**
- Lock 타임아웃을 반드시 설정
- Lock 해제를 `finally` 블록에서 보장

**성능 영향:**
- Lock 대기 시간이 길면 전체 응답 시간 증가
- 너무 많은 Lock 경합 발생 시 성능 저하

### 11.2 확률적 TTL의 한계

- **완벽한 스탬피드 방지 불가**: 여전히 동시 만료 가능
- **캐시 효율 감소**: TTL이 짧아지면 캐시 효율 감소
- **예측 어려움**: 정확한 만료 시점 예측 불가

### 11.3 백그라운드 갱신의 제약

- **예측 가능한 트래픽에만 효과적**: 예상치 못한 트래픽에는 효과 적음
- **추가 리소스 사용**: 백그라운드 작업으로 인한 리소스 소모
- **데이터 일관성**: 갱신 중 일시적 불일치 가능

---

## 12. 실제 시나리오별 적용

### 12.1 전자상거래 인기 상품

**특징:**
- 예측 가능한 트래픽 (특정 시간대 집중)
- 동일한 상품 정보를 많은 사용자가 조회

**권장 전략:**
- Lock + Probabilistic TTL + Background Refresh + Cache Warming

### 12.2 실시간 랭킹 정보

**특징:**
- 자주 갱신되는 데이터
- 많은 사용자가 동시 조회

**권장 전략:**
- Lock + Background Refresh (짧은 주기)

### 12.3 사용자 프로필 정보

**특징:**
- 사용자별로 다른 데이터
- 상대적으로 안정적인 데이터

**권장 전략:**
- Lock + Probabilistic TTL

---

## 13. 문제 해결 체크리스트

### 캐시 스탬피드 발생 시 점검 사항

- [ ] Lock 타임아웃이 적절한가?
- [ ] Double-Check 패턴이 적용되어 있는가?
- [ ] 캐시 TTL이 너무 긴가? (동시 만료 위험)
- [ ] 확률적 TTL이 적용되어 있는가?
- [ ] 백그라운드 갱신이 동작하는가?
- [ ] 캐시 워밍업이 실행되고 있는가?
- [ ] 모니터링이 제대로 설정되어 있는가?

### 성능 최적화 체크리스트

- [ ] 캐시 히트율이 95% 이상인가?
- [ ] 평균 응답 시간이 목표 이내인가?
- [ ] DB 부하가 정상 범위 내인가?
- [ ] Lock 경합이 과도하지 않은가?

---

## 마무리

**캐시 스탬피드는 분산 시스템에서 매우 흔한 문제**이며, 특히 **캐시가 만료되는 시점에 동시에 많은 요청이 들어올 때** 심각해집니다. Redis를 활용한 **Lock 기반 해결책**이 가장 직접적이고 효과적이며, **Probabilistic Early Expiration**과 **Background Refresh**를 함께 사용하면 더욱 안정적인 시스템을 구축할 수 있습니다.

**핵심 정리:**

1. **Lock 기반**: 동시 DB 조회를 방지하는 가장 효과적인 방법
2. **Probabilistic TTL**: 간단한 추가 보호 계층
3. **Background Refresh**: 예측 가능한 트래픽에 효과적
4. **Cache Warming**: 서비스 시작 시 필수
5. **모니터링**: 문제 조기 발견 및 대응

**다음 글에서는 JPA, TypeORM, Django ORM에서 N+1 문제를 해결하는 방식의 차이점을 비교해보겠습니다.** 🚀
