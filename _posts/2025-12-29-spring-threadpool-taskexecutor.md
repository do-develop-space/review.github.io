---
layout: post
title: "Spring ThreadPoolTaskExecutor: In-Process 비동기 처리 완전 가이드"
date: 2025-12-29
categories: [programming, spring, performance]
tags: [Spring, ThreadPoolTaskExecutor, 비동기처리, Async, ExecutorService, ThreadPool, In-Process]
---

# Spring ThreadPoolTaskExecutor: In-Process 비동기 처리 완전 가이드

이전 글에서 RAG 시스템 설계를 다뤘는데, 이번에는 RAG 시스템의 백엔드 구현과 관련하여 **Spring의 in-process 비동기 처리**와 **ThreadPoolTaskExecutor**를 활용한 비동기 처리 전략을 정리해보겠습니다.

RAG 시스템에서 임베딩 생성, 벡터 검색, LLM 호출 등은 모두 시간이 오래 걸리는 작업입니다. 이러한 작업들을 **동기적으로 처리하면 응답 시간이 길어지고, 사용자 경험이 나빠집니다.** 

Spring의 `@Async`와 `ThreadPoolTaskExecutor`를 활용하면 **같은 프로세스 내에서 비동기로 작업을 처리**할 수 있습니다.

---

## 1. In-Process 비동기 처리란?

### 개념

**In-Process 비동기 처리**는 **같은 JVM 프로세스 내에서 별도의 스레드를 생성하여 작업을 비동기로 처리**하는 방식입니다.

```
동기 처리:
요청 → [작업1] → [작업2] → [작업3] → 응답 (총 3초)

비동기 처리:
요청 → [작업1, 작업2, 작업3 병렬 실행] → 응답 (총 1초)
```

### Out-of-Process vs In-Process

| 구분 | Out-of-Process | In-Process |
|------|---------------|------------|
| **예시** | 메시지 큐(Kafka, RabbitMQ), 외부 서비스 호출 | ThreadPool, @Async |
| **통신** | 네트워크/IPC | 메모리 공유 |
| **지연 시간** | 높음 (네트워크 오버헤드) | 낮음 (메모리 접근) |
| **복잡도** | 높음 (인프라 구성 필요) | 낮음 (코드만 추가) |
| **사용 사례** | 서비스 간 통신, 장기 작업 | 빠른 비동기 처리, 내부 작업 |

**In-Process가 적합한 경우:**
- ✅ 같은 서비스 내에서 빠르게 처리해야 하는 작업
- ✅ 외부 인프라 없이 비동기 처리가 필요한 경우
- ✅ 작업 시간이 수 초 ~ 수십 초 정도인 경우

---

## 2. Spring @Async 기본 사용법

### 2.1 기본 설정

```java
@Configuration
@EnableAsync  // ← @Async 활성화
public class AsyncConfig {
    
    @Bean(name = "taskExecutor")
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);           // 기본 스레드 수
        executor.setMaxPoolSize(10);           // 최대 스레드 수
        executor.setQueueCapacity(100);        // 큐 크기
        executor.setThreadNamePrefix("async-"); // 스레드 이름 prefix
        executor.initialize();
        return executor;
    }
}
```

### 2.2 @Async 사용 예시

```java
@Service
public class RecipeRAGService {
    
    @Async("taskExecutor")  // ← 비동기 실행
    public CompletableFuture<List<RecipeChunk>> searchRecipes(String query) {
        // 벡터 검색 (시간이 오래 걸림)
        List<RecipeChunk> chunks = vectorSearch(query);
        return CompletableFuture.completedFuture(chunks);
    }
    
    @Async("taskExecutor")
    public CompletableFuture<String> generateEmbedding(String text) {
        // 임베딩 생성 (시간이 오래 걸림)
        String embedding = embeddingService.embed(text);
        return CompletableFuture.completedFuture(embedding);
    }
    
    @Async("taskExecutor")
    public CompletableFuture<String> callLLM(String prompt) {
        // LLM 호출 (시간이 오래 걸림)
        String response = llmService.generate(prompt);
        return CompletableFuture.completedFuture(response);
    }
}
```

### 2.3 비동기 작업 조합

```java
@Service
public class RecipeRAGService {
    
    public String answer(String userQuery) {
        // 1. 임베딩 생성 (비동기)
        CompletableFuture<float[]> embeddingFuture = 
            generateEmbedding(userQuery);
        
        // 2. 임베딩이 완료되면 벡터 검색 (비동기)
        CompletableFuture<List<RecipeChunk>> searchFuture = 
            embeddingFuture.thenCompose(embedding -> 
                searchRecipes(embedding)
            );
        
        // 3. 검색 결과로 프롬프트 구성 후 LLM 호출 (비동기)
        CompletableFuture<String> answerFuture = 
            searchFuture.thenCompose(chunks -> {
                String prompt = buildPrompt(chunks, userQuery);
                return callLLM(prompt);
            });
        
        // 4. 모든 작업 완료 대기
        return answerFuture.join();  // 또는 .get()
    }
}
```

---

## 3. ThreadPoolTaskExecutor 상세 설정

### 3.1 핵심 파라미터 이해

```java
@Bean
public ThreadPoolTaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    
    // 1. Core Pool Size (기본 스레드 수)
    // 항상 유지되는 스레드 수
    executor.setCorePoolSize(5);
    
    // 2. Max Pool Size (최대 스레드 수)
    // 큐가 가득 찬 후에만 생성되는 추가 스레드 수
    executor.setMaxPoolSize(10);
    
    // 3. Queue Capacity (큐 크기)
    // Core Pool이 가득 찰 때까지 대기하는 작업 수
    executor.setQueueCapacity(100);
    
    // 4. Keep Alive Seconds
    // Core Pool을 초과한 스레드의 유지 시간
    executor.setKeepAliveSeconds(60);
    
    // 5. Rejected Execution Handler
    // 큐와 Max Pool이 모두 가득 찼을 때의 처리 전략
    executor.setRejectedExecutionHandler(
        new ThreadPoolExecutor.CallerRunsPolicy()
    );
    
    executor.setThreadNamePrefix("async-");
    executor.initialize();
    return executor;
}
```

### 3.2 스레드 풀 동작 원리

```
작업 도착 순서:
1. Core Pool (5개)에 여유가 있으면 → 즉시 실행
2. Core Pool이 가득 차면 → Queue (100개)에 대기
3. Queue가 가득 차면 → Max Pool (10개)까지 스레드 추가 생성
4. Max Pool까지 가득 차면 → RejectedExecutionHandler 실행
```

**예시:**
```
현재 상태: Core Pool 5개 모두 사용 중, Queue 100개 대기 중

새 작업 도착:
- Core Pool: 5개 사용 중
- Queue: 100개 대기 중
- → Max Pool까지 스레드 추가 생성 (최대 10개)
- → 총 15개 작업 동시 실행 가능
```

### 3.3 RejectedExecutionHandler 전략

```java
// 1. AbortPolicy (기본값)
// 예외 발생 (RejectedExecutionException)
executor.setRejectedExecutionHandler(
    new ThreadPoolExecutor.AbortPolicy()
);

// 2. CallerRunsPolicy
// 호출한 스레드에서 직접 실행 (블로킹)
executor.setRejectedExecutionHandler(
    new ThreadPoolExecutor.CallerRunsPolicy()
);

// 3. DiscardPolicy
// 작업을 조용히 버림
executor.setRejectedExecutionHandler(
    new ThreadPoolExecutor.DiscardPolicy()
);

// 4. DiscardOldestPolicy
// 가장 오래된 작업을 버리고 새 작업 추가
executor.setRejectedExecutionHandler(
    new ThreadPoolExecutor.DiscardOldestPolicy()
);
```

**추천: CallerRunsPolicy**
- 시스템이 과부하일 때 호출 스레드를 블로킹하여 부하 조절
- 작업 손실 없음

---

## 4. 실전 사용 패턴

### 4.1 병렬 작업 실행

```java
@Service
public class RecipeRAGService {
    
    public RecipeResponse getRecipeWithDetails(Long recipeId) {
        // 여러 작업을 병렬로 실행
        CompletableFuture<Recipe> recipeFuture = 
            getRecipeAsync(recipeId);
        CompletableFuture<List<Review>> reviewsFuture = 
            getReviewsAsync(recipeId);
        CompletableFuture<NutritionInfo> nutritionFuture = 
            getNutritionAsync(recipeId);
        
        // 모든 작업 완료 대기
        CompletableFuture.allOf(
            recipeFuture, 
            reviewsFuture, 
            nutritionFuture
        ).join();
        
        return RecipeResponse.builder()
            .recipe(recipeFuture.join())
            .reviews(reviewsFuture.join())
            .nutrition(nutritionFuture.join())
            .build();
    }
    
    @Async("taskExecutor")
    private CompletableFuture<Recipe> getRecipeAsync(Long id) {
        return CompletableFuture.completedFuture(
            recipeRepository.findById(id).orElseThrow()
        );
    }
    
    @Async("taskExecutor")
    private CompletableFuture<List<Review>> getReviewsAsync(Long id) {
        return CompletableFuture.completedFuture(
            reviewRepository.findByRecipeId(id)
        );
    }
    
    @Async("taskExecutor")
    private CompletableFuture<NutritionInfo> getNutritionAsync(Long id) {
        return CompletableFuture.completedFuture(
            nutritionService.getNutrition(id)
        );
    }
}
```

### 4.2 타임아웃 설정

```java
@Service
public class RecipeRAGService {
    
    public String answerWithTimeout(String query) {
        CompletableFuture<String> answerFuture = 
            generateAnswerAsync(query);
        
        try {
            // 5초 타임아웃
            return answerFuture.get(5, TimeUnit.SECONDS);
        } catch (TimeoutException e) {
            // 타임아웃 시 기본 응답 반환
            answerFuture.cancel(true);  // 작업 취소
            return "응답 생성 중입니다. 잠시 후 다시 시도해주세요.";
        } catch (ExecutionException | InterruptedException e) {
            throw new RuntimeException("답변 생성 실패", e);
        }
    }
    
    @Async("taskExecutor")
    private CompletableFuture<String> generateAnswerAsync(String query) {
        // LLM 호출 등 시간이 오래 걸리는 작업
        return CompletableFuture.completedFuture(
            llmService.generate(query)
        );
    }
}
```

### 4.3 예외 처리

```java
@Service
public class RecipeRAGService {
    
    public String answerWithErrorHandling(String query) {
        CompletableFuture<String> answerFuture = 
            generateAnswerAsync(query)
                .exceptionally(throwable -> {
                    // 예외 발생 시 기본 응답 반환
                    log.error("답변 생성 실패", throwable);
                    return "죄송합니다. 답변을 생성하는 중 오류가 발생했습니다.";
                });
        
        return answerFuture.join();
    }
    
    @Async("taskExecutor")
    private CompletableFuture<String> generateAnswerAsync(String query) {
        try {
            return CompletableFuture.completedFuture(
                llmService.generate(query)
            );
        } catch (Exception e) {
            // CompletableFuture에서 예외를 전파
            CompletableFuture<String> future = new CompletableFuture<>();
            future.completeExceptionally(e);
            return future;
        }
    }
}
```

### 4.4 여러 ThreadPool 사용

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    
    // 빠른 작업용 (작은 스레드 풀)
    @Bean(name = "fastExecutor")
    public ThreadPoolTaskExecutor fastExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(5);
        executor.setQueueCapacity(50);
        executor.setThreadNamePrefix("fast-");
        executor.initialize();
        return executor;
    }
    
    // 느린 작업용 (큰 스레드 풀)
    @Bean(name = "slowExecutor")
    public ThreadPoolTaskExecutor slowExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("slow-");
        executor.initialize();
        return executor;
    }
}

@Service
public class RecipeRAGService {
    
    // 빠른 작업 (DB 조회 등)
    @Async("fastExecutor")
    public CompletableFuture<Recipe> getRecipe(Long id) {
        return CompletableFuture.completedFuture(
            recipeRepository.findById(id).orElseThrow()
        );
    }
    
    // 느린 작업 (LLM 호출 등)
    @Async("slowExecutor")
    public CompletableFuture<String> generateAnswer(String query) {
        return CompletableFuture.completedFuture(
            llmService.generate(query)
        );
    }
}
```

---

## 5. ThreadPoolTaskExecutor 모니터링

### 5.1 스레드 풀 상태 확인

```java
@Service
public class ThreadPoolMonitor {
    
    @Autowired
    @Qualifier("taskExecutor")
    private ThreadPoolTaskExecutor executor;
    
    public ThreadPoolStatus getStatus() {
        ThreadPoolExecutor threadPool = executor.getThreadPoolExecutor();
        
        return ThreadPoolStatus.builder()
            .corePoolSize(threadPool.getCorePoolSize())
            .maxPoolSize(threadPool.getMaximumPoolSize())
            .activeThreads(threadPool.getActiveCount())
            .queueSize(threadPool.getQueue().size())
            .completedTasks(threadPool.getCompletedTaskCount())
            .totalTasks(threadPool.getTaskCount())
            .build();
    }
}
```

### 5.2 Actuator를 통한 모니터링

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,threaddump
  metrics:
    export:
      prometheus:
        enabled: true
```

```java
@RestController
@RequestMapping("/actuator/threadpool")
public class ThreadPoolActuatorEndpoint {
    
    @Autowired
    @Qualifier("taskExecutor")
    private ThreadPoolTaskExecutor executor;
    
    @GetMapping("/status")
    public Map<String, Object> getStatus() {
        ThreadPoolExecutor threadPool = executor.getThreadPoolExecutor();
        
        return Map.of(
            "corePoolSize", threadPool.getCorePoolSize(),
            "maxPoolSize", threadPool.getMaximumPoolSize(),
            "activeThreads", threadPool.getActiveCount(),
            "queueSize", threadPool.getQueue().size(),
            "completedTasks", threadPool.getCompletedTaskCount()
        );
    }
}
```

### 5.3 알림 설정

```java
@Component
public class ThreadPoolMonitor {
    
    @Scheduled(fixedRate = 10000)  // 10초마다 체크
    public void monitorThreadPool() {
        ThreadPoolExecutor threadPool = executor.getThreadPoolExecutor();
        
        int queueSize = threadPool.getQueue().size();
        int maxQueueCapacity = 100;
        double queueUsage = (double) queueSize / maxQueueCapacity;
        
        // 큐 사용률이 80% 이상이면 알림
        if (queueUsage > 0.8) {
            log.warn("ThreadPool 큐 사용률이 높습니다: {}%", queueUsage * 100);
            // 알림 서비스 호출 (Slack, Email 등)
            notificationService.sendAlert("ThreadPool 큐 사용률 경고", queueUsage);
        }
    }
}
```

---

## 6. 주의사항과 베스트 프랙티스

### 6.1 @Async 주의사항

**문제 1: 같은 클래스 내에서 @Async 호출 불가**

```java
@Service
public class RecipeService {
    
    public void process() {
        // ❌ 같은 클래스 내에서 호출하면 비동기로 동작하지 않음
        asyncMethod();  // 동기 실행됨
    }
    
    @Async
    public void asyncMethod() {
        // 비동기로 실행되지 않음
    }
}

// ✅ 해결책: 별도 서비스로 분리
@Service
public class RecipeAsyncService {
    
    @Async
    public void asyncMethod() {
        // 정상적으로 비동기 실행됨
    }
}
```

**문제 2: void 메서드의 예외 처리**

```java
@Async
public void asyncMethod() {
    try {
        // 작업 수행
    } catch (Exception e) {
        // 예외를 로깅하거나 별도 처리
        log.error("비동기 작업 실패", e);
        // 예외가 호출자에게 전파되지 않음
    }
}

// ✅ 해결책: CompletableFuture 반환
@Async
public CompletableFuture<Void> asyncMethod() {
    try {
        // 작업 수행
        return CompletableFuture.completedFuture(null);
    } catch (Exception e) {
        CompletableFuture<Void> future = new CompletableFuture<>();
        future.completeExceptionally(e);
        return future;
    }
}
```

### 6.2 ThreadPool 크기 설정 가이드

**CPU 집약적 작업:**
```java
// CPU 코어 수 기반
int corePoolSize = Runtime.getRuntime().availableProcessors();
executor.setCorePoolSize(corePoolSize);
executor.setMaxPoolSize(corePoolSize * 2);
```

**I/O 집약적 작업 (DB, HTTP 호출 등):**
```java
// CPU 코어 수보다 훨씬 큰 스레드 풀
int corePoolSize = Runtime.getRuntime().availableProcessors() * 4;
executor.setCorePoolSize(corePoolSize);
executor.setMaxPoolSize(corePoolSize * 2);
```

**혼합 작업:**
```java
// 작업 유형에 따라 여러 ThreadPool 사용
// - fastExecutor: CPU 집약적 (작은 풀)
// - slowExecutor: I/O 집약적 (큰 풀)
```

### 6.3 메모리 고려사항

**문제: 스레드 풀이 너무 크면 메모리 부족**

```java
// 각 스레드는 기본적으로 1MB 스택 메모리 사용
// 100개 스레드 = 약 100MB 스택 메모리

// ✅ 해결책: 스택 크기 조정
executor.setThreadNamePrefix("async-");
executor.setWaitForTasksToCompleteOnShutdown(true);
executor.setAwaitTerminationSeconds(60);
```

### 6.4 Graceful Shutdown

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    
    @Bean
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        // ... 설정 ...
        
        // Graceful Shutdown 설정
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(60);  // 최대 60초 대기
        
        executor.initialize();
        return executor;
    }
    
    @PreDestroy
    public void shutdown() {
        taskExecutor().shutdown();
    }
}
```

---

## 7. RAG 시스템에 적용 예시

### 7.1 전체 파이프라인 비동기화

```java
@Service
public class RecipeRAGService {
    
    public CompletableFuture<String> answerAsync(String userQuery) {
        // 1. 임베딩 생성 (비동기)
        CompletableFuture<float[]> embeddingFuture = 
            generateEmbeddingAsync(userQuery);
        
        // 2. 임베딩 완료 후 벡터 검색 (비동기)
        CompletableFuture<List<RecipeChunk>> searchFuture = 
            embeddingFuture.thenComposeAsync(embedding -> 
                searchRecipesAsync(embedding), 
                taskExecutor  // 다른 스레드 풀 사용 가능
            );
        
        // 3. 검색 결과로 프롬프트 구성 후 LLM 호출 (비동기)
        CompletableFuture<String> answerFuture = 
            searchFuture.thenComposeAsync(chunks -> {
                String prompt = buildPrompt(chunks, userQuery);
                return callLLMAsync(prompt);
            });
        
        return answerFuture;
    }
    
    @Async("taskExecutor")
    private CompletableFuture<float[]> generateEmbeddingAsync(String text) {
        return CompletableFuture.completedFuture(
            embeddingService.embed(text)
        );
    }
    
    @Async("taskExecutor")
    private CompletableFuture<List<RecipeChunk>> searchRecipesAsync(float[] embedding) {
        return CompletableFuture.completedFuture(
            vectorSearchService.search(embedding, 5)
        );
    }
    
    @Async("taskExecutor")
    private CompletableFuture<String> callLLMAsync(String prompt) {
        return CompletableFuture.completedFuture(
            llmService.generate(prompt)
        );
    }
}
```

### 7.2 스트리밍 응답 (Server-Sent Events)

```java
@RestController
public class RecipeRAGController {
    
    @GetMapping(value = "/rag/answer", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> answerStream(@RequestParam String query) {
        return Flux.create(sink -> {
            recipeRAGService.answerAsync(query)
                .thenAccept(answer -> {
                    // 스트리밍으로 답변 전송
                    sink.next("data: " + answer + "\n\n");
                    sink.complete();
                })
                .exceptionally(throwable -> {
                    sink.error(throwable);
                    return null;
                });
        });
    }
}
```

---

## 마무리

**핵심 포인트:**

- **Spring의 `@Async`와 `ThreadPoolTaskExecutor`를 사용하면 같은 프로세스 내에서 비동기 처리가 가능합니다.**
- **ThreadPool 크기는 작업 유형(CPU 집약적 vs I/O 집약적)에 따라 다르게 설정해야 합니다.**
- **여러 작업을 병렬로 실행하거나 순차적으로 연결할 때 `CompletableFuture`를 활용하면 효율적입니다.**
- **스레드 풀 상태를 모니터링하고, Graceful Shutdown을 설정하여 안정적인 운영이 필요합니다.**

In-Process 비동기 처리는 **외부 인프라 없이 빠르게 비동기 처리를 구현**할 수 있는 강력한 도구입니다. RAG 시스템처럼 여러 단계의 작업을 순차/병렬로 처리해야 하는 경우에 특히 유용합니다.

다음 글에서는 이러한 비동기 처리와 관련된 **추가 주제**를 정리해볼 예정입니다. 🚀


