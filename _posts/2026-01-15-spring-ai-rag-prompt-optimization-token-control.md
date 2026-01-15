---
layout: post
title: "Spring AI RAG 프롬프트 최적화 및 토큰 사용 제어: 효율적인 LLM 호출 전략"
date: 2026-01-15
categories: [ai, spring, architecture]
tags: [Spring AI, RAG, 프롬프트엔지니어링, 토큰최적화, LLM, PromptTemplate, TokenUsage]
---

이전 글에서 RAG 시스템의 기본 설계와 서빙 사이즈 분류를 다뤘습니다. 이번 글에서는 **Spring AI를 활용한 RAG 환경에서 프롬프트 최적화와 토큰 사용량을 효과적으로 제어하는 방법**을 정리해보겠습니다.

LLM 호출 시 프롬프트 구성과 토큰 사용량 관리는 **비용, 성능, 응답 품질**에 직접적인 영향을 미칩니다. 특히 RAG 시스템에서는 검색된 문서를 프롬프트에 포함시키기 때문에 토큰 사용량이 급증할 수 있어, 효과적인 제어가 필수입니다.

---

## 1. Spring AI에서 프롬프트와 토큰의 관계

### 1.1 프롬프트 구성 요소

**Spring AI RAG에서의 프롬프트 구조:**

```java
@RestController
public class RecipeRagController {
    
    @Autowired
    private ChatClient chatClient;
    
    @Autowired
    private VectorStore vectorStore;
    
    public String ask(String question) {
        // 1. 검색 (Retrieval)
        List<Document> documents = vectorStore.similaritySearch(question);
        
        // 2. 프롬프트 구성 (Augmentation)
        String context = documents.stream()
            .map(Document::getContent)
            .collect(Collectors.joining("\n\n"));
        
        String prompt = String.format("""
            다음 레시피 정보를 바탕으로 질문에 답변해주세요.
            
            [레시피 정보]
            %s
            
            [질문]
            %s
            
            [요구사항]
            - 정확하고 간결하게 답변해주세요.
            - 레시피 정보에 없는 내용은 추측하지 마세요.
            """, context, question);
        
        // 3. LLM 호출 (Generation)
        return chatClient.call(prompt);
    }
}
```

**프롬프트 구성 요소:**
- **시스템 프롬프트**: LLM의 역할과 행동 지침
- **컨텍스트**: 검색된 문서들 (RAG의 핵심)
- **사용자 질문**: 실제 질문
- **지시사항**: 출력 형식, 제약사항 등

### 1.2 토큰 사용량의 영향

**토큰 사용량이 중요한 이유:**

1. **비용 (Cost)**
   - 대부분의 LLM은 토큰 수 기반 과금
   - 입력 토큰 + 출력 토큰 모두 비용 발생
   - 예: GPT-4 기준 입력 $0.03/1K tokens, 출력 $0.06/1K tokens

2. **성능 (Performance)**
   - 프롬프트가 길수록 처리 시간 증가
   - 컨텍스트 창 제한 (예: GPT-4 Turbo 128K tokens)

3. **응답 품질 (Quality)**
   - 너무 많은 컨텍스트 → 핵심 정보가 묻힘
   - 너무 적은 컨텍스트 → 관련 정보 부족

---

## 2. Spring AI PromptTemplate 활용

### 2.1 PromptTemplate 기본 사용

**Spring AI의 PromptTemplate:**

```java
@Service
public class RecipePromptService {
    
    private final PromptTemplate promptTemplate;
    
    public RecipePromptService() {
        // 프롬프트 템플릿 정의
        this.promptTemplate = new PromptTemplate("""
            당신은 전문 요리사입니다. 다음 레시피 정보를 바탕으로 질문에 답변해주세요.
            
            [레시피 정보]
            {context}
            
            [질문]
            {question}
            
            [답변 형식]
            - 정확하고 간결하게 답변해주세요.
            - 레시피 정보에 없는 내용은 추측하지 마세요.
            - 필요한 경우 재료와 조리 단계를 구체적으로 설명해주세요.
            """);
    }
    
    public String createPrompt(String context, String question) {
        Map<String, Object> variables = Map.of(
            "context", context,
            "question", question
        );
        return promptTemplate.render(variables);
    }
}
```

**장점:**
- ✅ 프롬프트 템플릿 재사용
- ✅ 변수 주입으로 유연성 확보
- ✅ 프롬프트 관리 용이

### 2.2 외부 파일 기반 템플릿 관리

**resources/prompts/recipe-prompt.st:**

```text
당신은 전문 요리사입니다. 다음 레시피 정보를 바탕으로 질문에 답변해주세요.

[레시피 정보]
{context}

[질문]
{question}

[답변 형식]
- 정확하고 간결하게 답변해주세요.
- 레시피 정보에 없는 내용은 추측하지 마세요.
- 필요한 경우 재료와 조리 단계를 구체적으로 설명해주세요.
```

**설정 클래스:**

```java
@Configuration
public class PromptTemplateConfig {
    
    @Bean
    public PromptTemplate recipePromptTemplate(ResourceLoader resourceLoader) throws IOException {
        Resource resource = resourceLoader.getResource("classpath:prompts/recipe-prompt.st");
        String template = StreamUtils.copyToString(resource.getInputStream(), StandardCharsets.UTF_8);
        return new PromptTemplate(template);
    }
}
```

**장점:**
- ✅ 코드와 프롬프트 분리
- ✅ 프롬프트 수정 시 재배포 불필요 (외부 파일 관리 시)
- ✅ 버전 관리 용이

---

## 3. 토큰 사용량 측정 및 제어

### 3.1 TokenUsage 추적

**Spring AI의 TokenUsage:**

```java
@Service
public class RecipeRagService {
    
    @Autowired
    private ChatClient chatClient;
    
    @Autowired
    private VectorStore vectorStore;
    
    public RAGResponse askWithTokenTracking(String question) {
        // 1. 검색
        List<Document> documents = vectorStore.similaritySearch(
            SearchRequest.query(question)
                .withTopK(5)  // 상위 5개만 검색
        );
        
        // 2. 프롬프트 구성
        String context = documents.stream()
            .map(Document::getContent)
            .collect(Collectors.joining("\n\n"));
        
        Prompt prompt = new Prompt(context, question);
        
        // 3. LLM 호출 (TokenUsage 추적)
        ChatResponse response = chatClient.callForResponse(prompt);
        
        // 4. 토큰 사용량 확인
        TokenUsage tokenUsage = response.getResult().getMetadata().getUsage();
        
        return RAGResponse.builder()
            .answer(response.getResult().getOutput().getContent())
            .inputTokens(tokenUsage.getPromptTokens())
            .outputTokens(tokenUsage.getGenerationTokens())
            .totalTokens(tokenUsage.getTotalTokens())
            .build();
    }
}
```

**TokenUsage 정보:**
- `promptTokens`: 입력 프롬프트 토큰 수
- `generationTokens`: 생성된 응답 토큰 수
- `totalTokens`: 총 토큰 수

### 3.2 토큰 사용량 로깅 및 모니터링

**AOP를 활용한 토큰 사용량 로깅:**

```java
@Aspect
@Component
@Slf4j
public class TokenUsageAspect {
    
    @Around("@annotation(TrackTokenUsage)")
    public Object trackTokenUsage(ProceedingJoinPoint joinPoint) throws Throwable {
        long startTime = System.currentTimeMillis();
        
        try {
            Object result = joinPoint.proceed();
            
            // TokenUsage 추출 (ChatResponse인 경우)
            if (result instanceof ChatResponse) {
                ChatResponse response = (ChatResponse) result;
                TokenUsage usage = response.getResult().getMetadata().getUsage();
                
                long duration = System.currentTimeMillis() - startTime;
                
                log.info("Token Usage - Input: {}, Output: {}, Total: {}, Duration: {}ms",
                    usage.getPromptTokens(),
                    usage.getGenerationTokens(),
                    usage.getTotalTokens(),
                    duration);
                
                // 메트릭 수집 (Prometheus 등)
                recordMetrics(usage, duration);
            }
            
            return result;
        } catch (Exception e) {
            log.error("LLM 호출 실패", e);
            throw e;
        }
    }
    
    private void recordMetrics(TokenUsage usage, long duration) {
        // Prometheus, Micrometer 등으로 메트릭 수집
        // 예: counter.increment("llm.tokens.input", usage.getPromptTokens());
    }
}
```

**사용 예시:**

```java
@Service
public class RecipeRagService {
    
    @TrackTokenUsage
    public String ask(String question) {
        // LLM 호출
        return chatClient.call(prompt);
    }
}
```

---

## 4. 컨텍스트 길이 제어 전략

### 4.1 검색 결과 개수 제한

**문제: 검색 결과가 많을수록 토큰 사용량 증가**

```java
@Service
public class RecipeRagService {
    
    @Autowired
    private VectorStore vectorStore;
    
    // ❌ 나쁜 예: 검색 결과를 모두 포함
    public String askBad(String question) {
        List<Document> documents = vectorStore.similaritySearch(
            SearchRequest.query(question)
                .withTopK(20)  // 너무 많은 문서
        );
        // 토큰 사용량 폭증
    }
    
    // ✅ 좋은 예: 필요한 만큼만 검색
    public String askGood(String question) {
        List<Document> documents = vectorStore.similaritySearch(
            SearchRequest.query(question)
                .withTopK(3)  // 상위 3개만 (질문 복잡도에 따라 조정)
                .withSimilarityThreshold(0.7)  // 유사도 임계값 설정
        );
    }
}
```

### 4.2 문서 청킹(Chunking) 최적화

**문서를 작은 청크로 나누기:**

```java
@Service
public class DocumentChunkingService {
    
    public List<String> chunkDocument(String document, int maxChunkSize) {
        // 텍스트를 maxChunkSize 토큰 크기로 청킹
        // 예: SentenceSplitter, TokenTextSplitter 사용
        
        TextSplitter splitter = new TokenTextSplitter(
            maxChunkSize,      // 최대 청크 크기 (토큰 수)
            maxChunkSize / 4   // 청크 간 겹침 (overlap)
        );
        
        return splitter.split(document);
    }
}
```

**청킹 전략:**
- **고정 크기 청킹**: 동일한 크기로 나누기 (단순하지만 의미 단위 무시)
- **의미 기반 청킹**: 문장, 단락 단위로 나누기 (더 자연스러움)
- **오버랩 청킹**: 청크 간 일부 겹치게 하기 (컨텍스트 연속성 유지)

### 4.3 컨텍스트 압축 및 요약

**긴 문서를 요약하여 포함:**

```java
@Service
public class ContextCompressionService {
    
    @Autowired
    private ChatClient chatClient;
    
    public String compressContext(List<Document> documents, int maxTokens) {
        // 1. 문서들을 합치기
        String fullContext = documents.stream()
            .map(Document::getContent)
            .collect(Collectors.joining("\n\n"));
        
        // 2. 토큰 수 확인
        int currentTokens = estimateTokens(fullContext);
        
        // 3. 최대 토큰 수 초과 시 요약
        if (currentTokens > maxTokens) {
            String summaryPrompt = String.format("""
                다음 레시피 정보를 핵심 내용만 남기고 요약해주세요.
                최대 %d 토큰 이하로 압축해주세요.
                
                [원본 정보]
                %s
                """, maxTokens, fullContext);
            
            return chatClient.call(summaryPrompt);
        }
        
        return fullContext;
    }
    
    private int estimateTokens(String text) {
        // 대략적인 토큰 수 추정 (예: 문자 수 / 4)
        return text.length() / 4;
    }
}
```

---

## 5. 프롬프트 최적화 기법

### 5.1 시스템 프롬프트 최적화

**효율적인 시스템 프롬프트 작성:**

```java
@Service
public class OptimizedPromptService {
    
    // ❌ 나쁜 예: 장황한 시스템 프롬프트
    private static final String BAD_SYSTEM_PROMPT = """
        당신은 매우 전문적이고 경험이 풍부한 요리사입니다.
        당신은 한국 요리, 양식, 일식, 중식 등 다양한 요리에 대한 깊은 지식을 가지고 있습니다.
        당신은 사용자에게 친절하고 정확한 정보를 제공하는 것을 목표로 합니다.
        당신은 레시피의 재료, 조리 방법, 요리 시간, 난이도 등 모든 측면에 대해 상세하게 설명할 수 있습니다.
        ...
        """;  // 너무 길어서 토큰 낭비
    
    // ✅ 좋은 예: 간결하고 핵심적인 시스템 프롬프트
    private static final String GOOD_SYSTEM_PROMPT = """
        당신은 전문 요리사입니다. 레시피 정보를 바탕으로 정확하고 간결하게 답변해주세요.
        """;  // 핵심만 담아서 토큰 절약
}
```

**시스템 프롬프트 최적화 원칙:**
- ✅ 핵심 역할과 목적만 명시
- ✅ 불필요한 설명 제거
- ✅ 구체적인 지시사항은 사용자 프롬프트로 이동

### 5.2 지시사항 구조화

**구조화된 지시사항:**

```java
@Service
public class StructuredPromptService {
    
    public String createStructuredPrompt(String context, String question) {
        return String.format("""
            [역할]
            전문 요리사
            
            [컨텍스트]
            %s
            
            [질문]
            %s
            
            [지시사항]
            1. 정확성: 레시피 정보에 있는 내용만 사용
            2. 간결성: 핵심만 간단히 설명
            3. 형식: 재료 → 조리 순서
            
            [제약사항]
            - 추측하지 않기
            - 없는 정보는 "정보 없음"으로 표시
            """, context, question);
    }
}
```

**장점:**
- ✅ LLM이 구조를 이해하기 쉬움
- ✅ 필요한 지시사항만 포함하여 토큰 절약
- ✅ 프롬프트 수정이 용이

### 5.3 Few-Shot Learning 활용

**예시를 포함한 프롬프트:**

```java
@Service
public class FewShotPromptService {
    
    private static final String FEW_SHOT_PROMPT = """
        [예시 1]
        질문: "김치찌개 1인분 재료 양 알려줘"
        답변: "돼지고기 100g, 김치 200g, 물 300ml, 대파 1대, 마늘 1쪽"
        
        [예시 2]
        질문: "된장찌개 조리 시간이 얼마나 걸려?"
        답변: "약 20-30분 정도 소요됩니다."
        
        [실제 질문]
        질문: {question}
        답변:
        """;
    
    public String createFewShotPrompt(String question) {
        Map<String, Object> variables = Map.of("question", question);
        return new PromptTemplate(FEW_SHOT_PROMPT).render(variables);
    }
}
```

**장점:**
- ✅ 원하는 출력 형식을 명확히 제시
- ✅ 토큰 사용량 증가하지만 응답 품질 향상
- ✅ 특정 형식의 출력이 필요한 경우 유용

---

## 6. 동적 토큰 제한 설정

### 6.1 질문 복잡도에 따른 동적 제한

```java
@Service
public class AdaptiveRagService {
    
    @Autowired
    private VectorStore vectorStore;
    
    public String askAdaptive(String question) {
        // 1. 질문 복잡도 분석
        QuestionComplexity complexity = analyzeComplexity(question);
        
        // 2. 복잡도에 따른 토큰 제한 설정
        TokenLimit limit = getTokenLimit(complexity);
        
        // 3. 검색 결과 개수 동적 조정
        int topK = complexity == QuestionComplexity.HIGH ? 5 : 3;
        
        // 4. 검색
        List<Document> documents = vectorStore.similaritySearch(
            SearchRequest.query(question).withTopK(topK)
        );
        
        // 5. 컨텍스트 압축
        String context = compressToLimit(documents, limit.getMaxContextTokens());
        
        // 6. 프롬프트 구성
        String prompt = createPrompt(context, question);
        
        // 7. LLM 호출 (최대 토큰 제한 설정)
        ChatOptions options = ChatOptions.builder()
            .withMaxTokens(limit.getMaxOutputTokens())
            .build();
        
        return chatClient.call(new Prompt(prompt, options));
    }
    
    private QuestionComplexity analyzeComplexity(String question) {
        // 질문 길이, 키워드 수 등으로 복잡도 판단
        int wordCount = question.split("\\s+").length;
        if (wordCount > 20) return QuestionComplexity.HIGH;
        if (wordCount > 10) return QuestionComplexity.MEDIUM;
        return QuestionComplexity.LOW;
    }
    
    private TokenLimit getTokenLimit(QuestionComplexity complexity) {
        return switch (complexity) {
            case HIGH -> new TokenLimit(8000, 2000);  // 컨텍스트 8K, 출력 2K
            case MEDIUM -> new TokenLimit(4000, 1000);  // 컨텍스트 4K, 출력 1K
            case LOW -> new TokenLimit(2000, 500);  // 컨텍스트 2K, 출력 500
        };
    }
}
```

### 6.2 ChatOptions를 통한 토큰 제어

```java
@Service
public class ControlledRagService {
    
    @Autowired
    private ChatClient chatClient;
    
    public String askWithControls(String context, String question) {
        // ChatOptions로 토큰 제어
        ChatOptions options = ChatOptions.builder()
            .withMaxTokens(1000)  // 최대 출력 토큰 수
            .withTemperature(0.7)  // 창의성 제어 (0.0 ~ 2.0)
            .withTopP(0.9)  // Top-p 샘플링
            .build();
        
        Prompt prompt = new Prompt(createPrompt(context, question), options);
        return chatClient.call(prompt);
    }
}
```

**ChatOptions 주요 파라미터:**
- `maxTokens`: 최대 출력 토큰 수
- `temperature`: 응답의 창의성/확정성 제어 (낮을수록 결정적)
- `topP`: Nucleus 샘플링 (다양성 제어)
- `topK`: Top-k 샘플링

---

## 7. 실제 서비스 환경 고려사항

### 7.1 비용 최적화

**토큰 사용량 모니터링 및 알림:**

```java
@Service
@Slf4j
public class CostOptimizedRagService {
    
    private final AtomicLong dailyTokenUsage = new AtomicLong(0);
    private static final long DAILY_TOKEN_LIMIT = 1_000_000;  // 하루 100만 토큰
    
    public String askWithCostControl(String question) {
        // 1. 일일 토큰 사용량 확인
        if (dailyTokenUsage.get() > DAILY_TOKEN_LIMIT) {
            throw new TokenLimitExceededException("일일 토큰 사용량 초과");
        }
        
        // 2. LLM 호출
        ChatResponse response = chatClient.callForResponse(prompt);
        
        // 3. 토큰 사용량 누적
        TokenUsage usage = response.getResult().getMetadata().getUsage();
        long total = dailyTokenUsage.addAndGet(usage.getTotalTokens());
        
        // 4. 임계값 초과 시 알림
        if (total > DAILY_TOKEN_LIMIT * 0.8) {
            log.warn("일일 토큰 사용량이 80%% 초과: {}", total);
            // 알림 발송 (Slack, Email 등)
        }
        
        return response.getResult().getOutput().getContent();
    }
}
```

### 7.2 캐싱 전략

**동일 질문에 대한 캐싱:**

```java
@Service
public class CachedRagService {
    
    @Autowired
    private Cache<String, String> answerCache;
    
    @Autowired
    private ChatClient chatClient;
    
    public String askCached(String question) {
        // 1. 캐시 확인
        String cached = answerCache.getIfPresent(question);
        if (cached != null) {
            log.debug("캐시에서 답변 반환: {}", question);
            return cached;
        }
        
        // 2. LLM 호출
        String answer = chatClient.call(prompt);
        
        // 3. 캐시 저장 (TTL: 1시간)
        answerCache.put(question, answer);
        
        return answer;
    }
}
```

**장점:**
- ✅ 동일 질문에 대한 토큰 사용량 절감
- ✅ 응답 시간 단축
- ✅ 비용 절감

### 7.3 배치 처리

**여러 질문을 한 번에 처리:**

```java
@Service
public class BatchRagService {
    
    @Autowired
    private ChatClient chatClient;
    
    public List<String> askBatch(List<String> questions) {
        // 배치 프롬프트 구성
        String batchPrompt = questions.stream()
            .map(q -> String.format("Q: %s", q))
            .collect(Collectors.joining("\n\n"));
        
        // 한 번의 호출로 여러 질문 처리
        String batchResponse = chatClient.call(batchPrompt);
        
        // 응답 파싱
        return parseBatchResponse(batchResponse);
    }
}
```

**주의사항:**
- ⚠️ 배치 크기가 너무 크면 토큰 제한 초과 가능
- ⚠️ 개별 질문 추적이 어려움
- ⚠️ 부분 실패 시 전체 재시도 필요

---

## 8. Best Practices

### 8.1 프롬프트 최적화 체크리스트

**✅ 시스템 프롬프트:**
- 핵심 역할과 목적만 명시
- 불필요한 설명 제거
- 일반적인 지시사항 포함

**✅ 컨텍스트 구성:**
- 검색 결과 개수 최소화 (3-5개)
- 관련성 높은 문서만 포함
- 필요시 압축/요약 적용

**✅ 사용자 질문:**
- 명확하고 구체적으로 작성
- 불필요한 설명 제거

**✅ 출력 형식:**
- 필요한 경우 Few-Shot 예시 포함
- 구조화된 지시사항 사용

### 8.2 토큰 사용량 모니터링

**모니터링 항목:**
- 일일/월간 토큰 사용량
- 평균 토큰 사용량 (요청당)
- 입력/출력 토큰 비율
- 비용 추적

**알림 설정:**
- 일일 토큰 사용량 80% 초과 시
- 비정상적인 토큰 사용량 감지 시
- 비용 예산 초과 시

### 8.3 점진적 최적화

**1단계: 기본 모니터링**
- 토큰 사용량 추적 및 로깅
- 비용 모니터링

**2단계: 프롬프트 최적화**
- 시스템 프롬프트 간소화
- 컨텍스트 길이 제어

**3단계: 고급 최적화**
- 동적 토큰 제한
- 캐싱 전략
- 배치 처리

---

## 마무리

**핵심 포인트:**

- **프롬프트 최적화는 토큰 사용량과 응답 품질의 균형을 찾는 과정입니다.**
- **Spring AI의 PromptTemplate와 ChatOptions를 활용하면 프롬프트와 토큰을 효과적으로 제어할 수 있습니다.**
- **RAG 시스템에서는 검색된 문서를 포함하기 때문에 컨텍스트 길이 제어가 특히 중요합니다.**
- **토큰 사용량 모니터링과 비용 최적화는 실제 서비스 환경에서 필수입니다.**

**최종 권장사항:**

- **프로덕션 환경**: 토큰 사용량 모니터링, 캐싱, 동적 제한 적용
- **개발 환경**: 프롬프트 최적화, 컨텍스트 압축 실험
- **일반적인 경우**: 검색 결과 3-5개, 최대 출력 토큰 1000-2000, 캐싱 적용

Spring AI를 활용한 RAG 시스템에서 프롬프트와 토큰을 효과적으로 제어하면, **비용을 절감하면서도 응답 품질을 유지**할 수 있습니다. 자신의 서비스 환경에서 토큰 사용량을 모니터링하고 지속적으로 최적화하는 것이 중요합니다. 🚀

