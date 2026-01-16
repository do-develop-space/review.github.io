---
layout: post
title: "RAG에서 낮은 유사도 검색 결과 처리: Fallback 전략과 알고리즘적 해결"
date: 2026-01-16
categories: [ai, search, architecture]
tags: [RAG, 유사도검색, Fallback, 벡터검색, 임계값, 하이브리드검색, QueryRewrite, Re-ranking]
---

이전 글에서 Spring AI RAG의 프롬프트 최적화와 토큰 사용량 제어를 다뤘습니다. 이번 글에서는 **벡터 검색에서 관련 문서를 찾지 못하거나 유사도가 너무 낮을 때 어떻게 처리할지**에 대한 전략을 정리해보겠습니다.

RAG 시스템에서 가장 큰 문제 중 하나는 **사용자 질문에 대한 관련 문서가 지식 베이스에 없을 때**입니다. 단순히 유사도가 낮은 문서를 제공하면 LLM이 잘못된 정보를 생성할 수 있어, 효과적인 Fallback 전략이 필수입니다.

---

## 1. 문제 상황: 낮은 유사도 검색 결과

### 1.1 문제 시나리오

**일반적인 RAG 흐름:**

```
사용자 질문: "파스타 레시피 알려줘"
    ↓
[벡터 검색] → 유사도 0.85, 0.82, 0.78 문서 반환
    ↓
[LLM] → 검색된 문서 기반으로 답변 생성 ✅
```

**문제 상황:**

```
사용자 질문: "우주 식당에서 먹을 수 있는 음식은?"
    ↓
[벡터 검색] → 유사도 0.35, 0.32, 0.28 문서 반환
    ↓
[LLM] → 관련 없는 문서 기반으로 답변 생성 ❌ (환각 발생)
```

### 1.2 문제의 본질

**벡터 검색의 한계:**

1. **유사도가 낮아도 결과 반환**
   - 벡터 검색은 항상 상위 k개 문서를 반환
   - 유사도가 0.3이어도 가장 유사한 문서로 선택됨

2. **관련 문서가 없을 때의 처리 부재**
   - 검색 결과가 없어도 빈 리스트 대신 낮은 유사도 문서 반환
   - LLM은 제공된 문서를 기반으로 답변을 생성하려 함

3. **도메인 외 질문 감지 어려움**
   - 레시피 RAG인데 "주식 투자 방법"을 물어볼 때
   - 벡터 검색은 여전히 낮은 유사도로 결과 반환

---

## 2. 알고리즘적 접근: 유사도 임계값 설정

### 2.1 유사도 임계값(Threshold) 기반 필터링

**기본 개념:**

벡터 검색 결과의 유사도 점수가 임계값보다 낮으면 검색 결과로 간주하지 않음

```java
@Service
public class ThresholdBasedRagService {
    
    @Autowired
    private VectorStore vectorStore;
    
    private static final double SIMILARITY_THRESHOLD = 0.7;  // 유사도 임계값
    
    public RAGResponse askWithThreshold(String question) {
        // 1. 벡터 검색
        List<Document> documents = vectorStore.similaritySearch(
            SearchRequest.query(question)
                .withTopK(5)
        );
        
        // 2. 유사도 임계값 필터링
        List<Document> filteredDocuments = documents.stream()
            .filter(doc -> doc.getMetadata().getSimilarityScore() >= SIMILARITY_THRESHOLD)
            .collect(Collectors.toList());
        
        // 3. 검색 결과가 없으면 Fallback
        if (filteredDocuments.isEmpty()) {
            return handleNoResults(question);
        }
        
        // 4. LLM 호출
        String answer = generateAnswer(filteredDocuments, question);
        return new RAGResponse(answer, filteredDocuments);
    }
    
    private RAGResponse handleNoResults(String question) {
        // Fallback 전략 (후술)
        return new RAGResponse("죄송합니다. 관련 레시피 정보를 찾을 수 없습니다.", Collections.emptyList());
    }
}
```

**장점:**
- ✅ 구현이 간단하고 직관적
- ✅ 명확한 기준으로 필터링 가능

**단점:**
- ❌ 임계값 설정이 어려움 (도메인, 모델에 따라 다름)
- ❌ 고정된 임계값은 다양한 질문 유형에 부적합할 수 있음

### 2.2 동적 임계값 설정

**질문 유형별 임계값 조정:**

```java
@Service
public class AdaptiveThresholdRagService {
    
    @Autowired
    private VectorStore vectorStore;
    
    public RAGResponse askAdaptive(String question) {
        // 1. 질문 유형 분석
        QuestionType questionType = analyzeQuestionType(question);
        
        // 2. 질문 유형별 임계값 설정
        double threshold = getThresholdByType(questionType);
        
        // 3. 벡터 검색
        List<Document> documents = vectorStore.similaritySearch(
            SearchRequest.query(question).withTopK(5)
        );
        
        // 4. 동적 임계값 필터링
        List<Document> filtered = documents.stream()
            .filter(doc -> doc.getMetadata().getSimilarityScore() >= threshold)
            .collect(Collectors.toList());
        
        if (filtered.isEmpty()) {
            return handleNoResults(question);
        }
        
        return generateAnswer(filtered, question);
    }
    
    private QuestionType analyzeQuestionType(String question) {
        // 질문 유형 분류 (예: 구체적 레시피 요청, 일반 질문 등)
        if (question.contains("레시피") || question.contains("만들기")) {
            return QuestionType.SPECIFIC_RECIPE;
        } else if (question.contains("뭐") || question.contains("어떤")) {
            return QuestionType.GENERAL_INQUIRY;
        }
        return QuestionType.OTHER;
    }
    
    private double getThresholdByType(QuestionType type) {
        return switch (type) {
            case SPECIFIC_RECIPE -> 0.75;  // 구체적 레시피 요청은 높은 임계값
            case GENERAL_INQUIRY -> 0.65;  // 일반 질문은 낮은 임계값
            case OTHER -> 0.70;
        };
    }
}
```

### 2.3 통계적 임계값 설정

**검색 결과 분포 기반 임계값:**

```java
@Service
public class StatisticalThresholdRagService {
    
    @Autowired
    private VectorStore vectorStore;
    
    public RAGResponse askStatistical(String question) {
        // 1. 벡터 검색 (상위 k개)
        List<Document> documents = vectorStore.similaritySearch(
            SearchRequest.query(question).withTopK(10)
        );
        
        // 2. 유사도 점수 추출
        List<Double> scores = documents.stream()
            .map(doc -> doc.getMetadata().getSimilarityScore())
            .collect(Collectors.toList());
        
        // 3. 통계적 임계값 계산
        double threshold = calculateStatisticalThreshold(scores);
        
        // 4. 필터링
        List<Document> filtered = documents.stream()
            .filter(doc -> doc.getMetadata().getSimilarityScore() >= threshold)
            .collect(Collectors.toList());
        
        if (filtered.isEmpty()) {
            return handleNoResults(question);
        }
        
        return generateAnswer(filtered, question);
    }
    
    private double calculateStatisticalThreshold(List<Double> scores) {
        if (scores.isEmpty()) return 0.7;
        
        // 방법 1: 평균 - 표준편차
        double mean = scores.stream().mapToDouble(Double::doubleValue).average().orElse(0.0);
        double variance = scores.stream()
            .mapToDouble(score -> Math.pow(score - mean, 2))
            .average().orElse(0.0);
        double stdDev = Math.sqrt(variance);
        
        double threshold1 = mean - stdDev;
        
        // 방법 2: 최고 점수와의 차이 기반
        double maxScore = scores.stream().mapToDouble(Double::doubleValue).max().orElse(0.0);
        double threshold2 = maxScore * 0.8;  // 최고 점수의 80%
        
        // 방법 3: 상위 점수 평균
        double threshold3 = scores.stream()
            .sorted(Collections.reverseOrder())
            .limit(3)
            .mapToDouble(Double::doubleValue)
            .average().orElse(0.0) * 0.9;
        
        // 가장 보수적인 임계값 선택
        return Math.max(Math.max(threshold1, threshold2), threshold3);
    }
}
```

---

## 3. 로직적 접근: Fallback 전략

### 3.1 다단계 검색 전략

**1차: 벡터 검색 → 2차: 키워드 검색 → 3차: Fallback**

```java
@Service
public class MultiStageRagService {
    
    @Autowired
    private VectorStore vectorStore;
    
    @Autowired
    private RecipeRepository recipeRepository;  // 키워드 검색용
    
    public RAGResponse askMultiStage(String question) {
        // 1단계: 벡터 검색 (임계값 0.7)
        List<Document> vectorResults = vectorStore.similaritySearch(
            SearchRequest.query(question)
                .withTopK(5)
                .withSimilarityThreshold(0.7)
        );
        
        if (!vectorResults.isEmpty()) {
            return generateAnswer(vectorResults, question);
        }
        
        // 2단계: 키워드 검색 (Fallback)
        List<Document> keywordResults = keywordSearch(question);
        
        if (!keywordResults.isEmpty()) {
            return generateAnswer(keywordResults, question);
        }
        
        // 3단계: 최종 Fallback
        return handleNoResults(question);
    }
    
    private List<Document> keywordSearch(String question) {
        // 키워드 추출
        Set<String> keywords = extractKeywords(question);
        
        // 키워드 기반 검색 (예: Elasticsearch match query)
        return recipeRepository.findByKeywords(keywords);
    }
    
    private Set<String> extractKeywords(String question) {
        // 간단한 키워드 추출 (예: 명사 추출, 불용어 제거)
        // 실제로는 형태소 분석기(NLP) 사용
        return Set.of(question.split("\\s+"));
    }
}
```

### 3.2 하이브리드 검색 (Hybrid Search)

**벡터 검색 + 키워드 검색 조합:**

```java
@Service
public class HybridSearchRagService {
    
    @Autowired
    private VectorStore vectorStore;
    
    @Autowired
    private ElasticsearchService elasticsearchService;
    
    public RAGResponse askHybrid(String question) {
        // 1. 벡터 검색
        List<Document> vectorResults = vectorStore.similaritySearch(
            SearchRequest.query(question).withTopK(10)
        );
        
        // 2. 키워드 검색
        List<Document> keywordResults = elasticsearchService.search(
            SearchRequest.builder()
                .query(QueryBuilders.matchQuery("content", question))
                .size(10)
                .build()
        );
        
        // 3. 결과 통합 (Reciprocal Rank Fusion)
        List<Document> mergedResults = mergeResults(vectorResults, keywordResults);
        
        // 4. 유사도 재계산
        List<Document> scoredResults = rescoreResults(mergedResults, question);
        
        // 5. 최종 필터링
        List<Document> filtered = scoredResults.stream()
            .filter(doc -> doc.getMetadata().getFinalScore() >= 0.6)
            .limit(5)
            .collect(Collectors.toList());
        
        if (filtered.isEmpty()) {
            return handleNoResults(question);
        }
        
        return generateAnswer(filtered, question);
    }
    
    private List<Document> mergeResults(List<Document> vector, List<Document> keyword) {
        // Reciprocal Rank Fusion (RRF) 방식으로 통합
        Map<String, Document> merged = new HashMap<>();
        
        // 벡터 검색 결과 점수 부여 (rank 기반)
        for (int i = 0; i < vector.size(); i++) {
            Document doc = vector.get(i);
            double rrfScore = 1.0 / (60 + i + 1);  // RRF 점수
            doc.getMetadata().put("vectorScore", rrfScore);
            merged.put(doc.getId(), doc);
        }
        
        // 키워드 검색 결과 점수 부여
        for (int i = 0; i < keyword.size(); i++) {
            Document doc = keyword.get(i);
            double rrfScore = 1.0 / (60 + i + 1);
            doc.getMetadata().put("keywordScore", rrfScore);
            
            // 이미 벡터 검색 결과에 있으면 점수 합산
            if (merged.containsKey(doc.getId())) {
                Document existing = merged.get(doc.getId());
                double combinedScore = (Double) existing.getMetadata().get("vectorScore") + rrfScore;
                existing.getMetadata().put("finalScore", combinedScore);
            } else {
                doc.getMetadata().put("finalScore", rrfScore);
                merged.put(doc.getId(), doc);
            }
        }
        
        return new ArrayList<>(merged.values());
    }
    
    private List<Document> rescoreResults(List<Document> results, String question) {
        // 최종 점수 기준 정렬
        return results.stream()
            .sorted((a, b) -> Double.compare(
                (Double) b.getMetadata().getOrDefault("finalScore", 0.0),
                (Double) a.getMetadata().getOrDefault("finalScore", 0.0)
            ))
            .collect(Collectors.toList());
    }
}
```

### 3.3 쿼리 리라이트(Query Rewrite)

**질문을 재구성하여 검색 품질 향상:**

```java
@Service
public class QueryRewriteRagService {
    
    @Autowired
    private ChatClient chatClient;  // LLM으로 쿼리 리라이트
    
    @Autowired
    private VectorStore vectorStore;
    
    public RAGResponse askWithRewrite(String question) {
        // 1. 원본 질문으로 검색
        List<Document> originalResults = vectorStore.similaritySearch(
            SearchRequest.query(question).withTopK(5)
        );
        
        // 2. 최고 유사도 확인
        double maxSimilarity = originalResults.stream()
            .mapToDouble(doc -> doc.getMetadata().getSimilarityScore())
            .max()
            .orElse(0.0);
        
        // 3. 유사도가 낮으면 쿼리 리라이트
        if (maxSimilarity < 0.7) {
            String rewrittenQuery = rewriteQuery(question);
            List<Document> rewrittenResults = vectorStore.similaritySearch(
                SearchRequest.query(rewrittenQuery).withTopK(5)
            );
            
            // 리라이트 결과가 더 좋으면 사용
            double rewrittenMaxSimilarity = rewrittenResults.stream()
                .mapToDouble(doc -> doc.getMetadata().getSimilarityScore())
                .max()
                .orElse(0.0);
            
            if (rewrittenMaxSimilarity > maxSimilarity) {
                originalResults = rewrittenResults;
            }
        }
        
        // 4. 최종 필터링
        List<Document> filtered = originalResults.stream()
            .filter(doc -> doc.getMetadata().getSimilarityScore() >= 0.65)
            .collect(Collectors.toList());
        
        if (filtered.isEmpty()) {
            return handleNoResults(question);
        }
        
        return generateAnswer(filtered, question);
    }
    
    private String rewriteQuery(String question) {
        // LLM으로 쿼리 리라이트
        String rewritePrompt = String.format("""
            다음 질문을 레시피 검색에 적합한 키워드로 재구성해주세요.
            핵심 키워드만 추출하여 간결하게 작성해주세요.
            
            원본 질문: %s
            재구성된 질문:
            """, question);
        
        return chatClient.call(rewritePrompt);
    }
}
```

**예시:**

```
원본 질문: "우주 식당에서 먹을 수 있는 음식은?"
리라이트: "우주 음식 레시피"

원본 질문: "파스타 만드는 법"
리라이트: "파스타 레시피"
```

### 3.4 Re-ranking 전략

**검색 결과 재순위화:**

```java
@Service
public class ReRankingRagService {
    
    @Autowired
    private VectorStore vectorStore;
    
    @Autowired
    private CrossEncoderService crossEncoderService;  // Cross-encoder 모델
    
    public RAGResponse askWithReRanking(String question) {
        // 1. 벡터 검색 (더 많은 결과 검색)
        List<Document> documents = vectorStore.similaritySearch(
            SearchRequest.query(question).withTopK(20)  // 더 많이 검색
        );
        
        // 2. Re-ranking (Cross-encoder 사용)
        List<Document> reranked = rerankDocuments(documents, question);
        
        // 3. 상위 k개 선택
        List<Document> topK = reranked.stream()
            .filter(doc -> doc.getMetadata().getRerankScore() >= 0.6)
            .limit(5)
            .collect(Collectors.toList());
        
        if (topK.isEmpty()) {
            return handleNoResults(question);
        }
        
        return generateAnswer(topK, question);
    }
    
    private List<Document> rerankDocuments(List<Document> documents, String question) {
        // Cross-encoder로 재점수화
        return documents.stream()
            .map(doc -> {
                double score = crossEncoderService.score(question, doc.getContent());
                doc.getMetadata().put("rerankScore", score);
                return doc;
            })
            .sorted((a, b) -> Double.compare(
                (Double) b.getMetadata().get("rerankScore"),
                (Double) a.getMetadata().get("rerankScore")
            ))
            .collect(Collectors.toList());
    }
}
```

**Cross-encoder vs Bi-encoder:**

- **Bi-encoder (벡터 검색)**: 질문과 문서를 각각 임베딩 → 빠르지만 정확도 낮음
- **Cross-encoder (Re-ranking)**: 질문과 문서를 함께 인코딩 → 느리지만 정확도 높음

**하이브리드 접근:**
1. Bi-encoder로 상위 20개 검색 (빠름)
2. Cross-encoder로 상위 20개 재순위화 (정확)
3. 최종 상위 5개 선택

---

## 4. 최종 Fallback 전략

### 4.1 사용자 피드백 기반 Fallback

**검색 결과 없을 때 적절한 응답:**

```java
@Service
public class FallbackRagService {
    
    @Autowired
    private ChatClient chatClient;
    
    public RAGResponse handleNoResults(String question) {
        // 1. 도메인 외 질문인지 확인
        boolean isOutOfDomain = isOutOfDomain(question);
        
        if (isOutOfDomain) {
            return new RAGResponse(
                "죄송합니다. 현재 레시피 관련 질문만 답변 가능합니다. " +
                "레시피 검색을 원하시면 구체적으로 질문해주세요.",
                Collections.emptyList(),
                RAGResponseType.OUT_OF_DOMAIN
            );
        }
        
        // 2. 관련 문서가 없는 경우
        return new RAGResponse(
            "죄송합니다. 요청하신 레시피 정보를 찾을 수 없습니다. " +
            "다른 레시피로 검색해보시겠어요?",
            Collections.emptyList(),
            RAGResponseType.NO_RESULTS
        );
    }
    
    private boolean isOutOfDomain(String question) {
        // 도메인 키워드 확인
        Set<String> domainKeywords = Set.of("레시피", "요리", "음식", "재료", "조리");
        Set<String> questionWords = Set.of(question.toLowerCase().split("\\s+"));
        
        // 도메인 키워드가 전혀 없으면 도메인 외로 판단
        return questionWords.stream().noneMatch(word -> 
            domainKeywords.stream().anyMatch(keyword -> word.contains(keyword))
        );
    }
}
```

### 4.2 일반 LLM 응답 Fallback

**RAG 실패 시 일반 LLM 응답:**

```java
@Service
public class LLMFallbackRagService {
    
    @Autowired
    private ChatClient chatClient;
    
    public RAGResponse askWithLLMFallback(String question) {
        // 1. 벡터 검색 시도
        List<Document> documents = searchDocuments(question);
        
        if (documents.isEmpty() || isLowSimilarity(documents)) {
            // 2. Fallback: 일반 LLM 응답
            return handleLLMFallback(question);
        }
        
        // 3. RAG 응답
        return generateRAGAnswer(documents, question);
    }
    
    private RAGResponse handleLLMFallback(String question) {
        String fallbackPrompt = String.format("""
            다음 질문에 대해 레시피 관련 답변을 해주세요.
            단, 확실하지 않은 내용은 "정확한 정보를 확인할 수 없습니다"라고 답변해주세요.
            
            질문: %s
            """, question);
        
        String answer = chatClient.call(fallbackPrompt);
        
        return new RAGResponse(
            answer,
            Collections.emptyList(),
            RAGResponseType.LLM_FALLBACK
        );
    }
}
```

**주의사항:**
- ⚠️ 일반 LLM 응답은 환각 가능성 높음
- ⚠️ 명확히 "일반 응답"임을 사용자에게 알려야 함
- ⚠️ 가능하면 사용하지 않는 것이 좋음

### 4.3 검색 히스토리 기반 개선

**과거 검색 결과 활용:**

```java
@Service
public class HistoryBasedRagService {
    
    @Autowired
    private VectorStore vectorStore;
    
    @Autowired
    private SearchHistoryRepository searchHistoryRepository;
    
    public RAGResponse askWithHistory(String question, String userId) {
        // 1. 현재 검색
        List<Document> currentResults = vectorStore.similaritySearch(
            SearchRequest.query(question).withTopK(5)
        );
        
        // 2. 검색 히스토리 확인
        List<SearchHistory> history = searchHistoryRepository.findByUserId(userId);
        
        // 3. 유사한 과거 질문이 있으면 해당 결과 활용
        Optional<SearchHistory> similarHistory = findSimilarHistory(question, history);
        
        if (similarHistory.isPresent() && isLowSimilarity(currentResults)) {
            // 과거 검색 결과 활용
            return generateAnswer(
                similarHistory.get().getDocuments(),
                question
            );
        }
        
        // 4. 일반 RAG 응답
        RAGResponse response = generateAnswer(currentResults, question);
        
        // 5. 히스토리 저장
        searchHistoryRepository.save(new SearchHistory(userId, question, currentResults));
        
        return response;
    }
}
```

---

## 5. 실제 구현 예시: 통합 전략

### 5.1 통합 RAG 서비스

**여러 전략을 조합한 최종 구현:**

```java
@Service
@Slf4j
public class IntegratedRagService {
    
    @Autowired
    private VectorStore vectorStore;
    
    @Autowired
    private ElasticsearchService elasticsearchService;
    
    @Autowired
    private ChatClient chatClient;
    
    private static final double VECTOR_THRESHOLD = 0.7;
    private static final double HYBRID_THRESHOLD = 0.6;
    
    public RAGResponse ask(String question) {
        try {
            // 1단계: 벡터 검색 (임계값 0.7)
            List<Document> vectorResults = vectorStore.similaritySearch(
                SearchRequest.query(question)
                    .withTopK(5)
                    .withSimilarityThreshold(VECTOR_THRESHOLD)
            );
            
            if (!vectorResults.isEmpty()) {
                log.info("벡터 검색 성공: {}개 결과", vectorResults.size());
                return generateAnswer(vectorResults, question);
            }
            
            // 2단계: 하이브리드 검색 (벡터 + 키워드)
            List<Document> hybridResults = hybridSearch(question);
            
            if (!hybridResults.isEmpty()) {
                log.info("하이브리드 검색 성공: {}개 결과", hybridResults.size());
                return generateAnswer(hybridResults, question);
            }
            
            // 3단계: 쿼리 리라이트 후 재검색
            String rewrittenQuery = rewriteQuery(question);
            List<Document> rewrittenResults = vectorStore.similaritySearch(
                SearchRequest.query(rewrittenQuery).withTopK(5)
            );
            
            double maxSimilarity = rewrittenResults.stream()
                .mapToDouble(doc -> doc.getMetadata().getSimilarityScore())
                .max()
                .orElse(0.0);
            
            if (maxSimilarity >= HYBRID_THRESHOLD) {
                log.info("쿼리 리라이트 후 검색 성공: 유사도 {}", maxSimilarity);
                return generateAnswer(rewrittenResults, question);
            }
            
            // 4단계: Fallback
            log.warn("모든 검색 실패, Fallback 실행");
            return handleNoResults(question);
            
        } catch (Exception e) {
            log.error("RAG 검색 중 오류 발생", e);
            return handleError(question, e);
        }
    }
    
    private List<Document> hybridSearch(String question) {
        // 벡터 검색
        List<Document> vectorResults = vectorStore.similaritySearch(
            SearchRequest.query(question).withTopK(10)
        );
        
        // 키워드 검색
        List<Document> keywordResults = elasticsearchService.search(question);
        
        // 결과 통합 (RRF)
        return mergeWithRRF(vectorResults, keywordResults);
    }
    
    private String rewriteQuery(String question) {
        String prompt = String.format("""
            다음 질문을 레시피 검색에 적합한 키워드로 재구성해주세요.
            핵심 키워드만 추출하여 간결하게 작성해주세요.
            
            원본: %s
            재구성:
            """, question);
        
        return chatClient.call(prompt);
    }
    
    private RAGResponse handleNoResults(String question) {
        // 도메인 외 질문 확인
        if (isOutOfDomain(question)) {
            return new RAGResponse(
                "죄송합니다. 레시피 관련 질문만 답변 가능합니다.",
                Collections.emptyList(),
                RAGResponseType.OUT_OF_DOMAIN
            );
        }
        
        // 일반 Fallback
        return new RAGResponse(
            "죄송합니다. 요청하신 레시피 정보를 찾을 수 없습니다.",
            Collections.emptyList(),
            RAGResponseType.NO_RESULTS
        );
    }
}
```

---

## 6. Best Practices

### 6.1 유사도 임계값 설정 가이드

**임계값 설정 원칙:**

1. **초기 설정**: 0.7 정도로 시작 (도메인에 따라 조정)
2. **평가**: 실제 질문-답변 쌍으로 평가
3. **조정**: False Positive/Negative 비율 확인 후 조정
4. **모니터링**: 검색 실패율, 사용자 피드백 모니터링

**도메인별 권장 임계값:**

- **의료/법률**: 0.8 이상 (높은 정확도 필요)
- **일반 지식/레시피**: 0.65-0.75
- **고객 지원/FAQ**: 0.6-0.7

### 6.2 전략 선택 가이드

**상황별 권장 전략:**

| 상황 | 권장 전략 | 이유 |
|------|----------|------|
| 검색 정확도가 중요한 경우 | 하이브리드 검색 + Re-ranking | 다양한 신호 활용 |
| 검색 속도가 중요한 경우 | 벡터 검색 + 고정 임계값 | 단순하고 빠름 |
| 도메인 외 질문이 많은 경우 | 도메인 필터링 + 쿼리 리라이트 | 질문 품질 개선 |
| 검색 실패율이 높은 경우 | 다단계 Fallback | 단계별 대응 |

### 6.3 모니터링 및 평가

**모니터링 지표:**

1. **검색 성공률**: 임계값 이상 검색 결과 비율
2. **Fallback 실행률**: Fallback 전략 실행 빈도
3. **사용자 만족도**: 답변 품질 평가
4. **평균 유사도**: 검색 결과의 평균 유사도 점수

**평가 방법:**

```java
@Service
public class RAGEvaluationService {
    
    public void evaluate(String question, List<Document> results, String expectedAnswer) {
        // 1. 검색 결과 유사도 확인
        double avgSimilarity = results.stream()
            .mapToDouble(doc -> doc.getMetadata().getSimilarityScore())
            .average()
            .orElse(0.0);
        
        // 2. 검색 결과 관련성 평가 (수동 또는 자동)
        boolean isRelevant = evaluateRelevance(results, expectedAnswer);
        
        // 3. 메트릭 수집
        recordMetrics(avgSimilarity, isRelevant);
    }
}
```

---

## 마무리

**핵심 포인트:**

- **벡터 검색은 항상 결과를 반환하므로, 유사도 임계값 설정이 필수입니다.**
- **알고리즘적 접근(임계값)과 로직적 접근(Fallback)을 조합하여 효과적인 전략을 수립할 수 있습니다.**
- **하이브리드 검색, 쿼리 리라이트, Re-ranking 등 다양한 기법을 상황에 맞게 활용해야 합니다.**
- **최종 Fallback 전략은 사용자에게 명확한 피드백을 제공해야 합니다.**

**최종 권장사항:**

- **프로덕션 환경**: 하이브리드 검색 + 동적 임계값 + 다단계 Fallback
- **초기 개발**: 벡터 검색 + 고정 임계값(0.7) + 기본 Fallback
- **고품질 요구**: Re-ranking + 쿼리 리라이트 + 통계적 임계값

RAG 시스템에서 낮은 유사도 검색 결과를 효과적으로 처리하면, **사용자에게 정확하고 신뢰할 수 있는 답변을 제공**할 수 있습니다. 자신의 서비스 환경에서 다양한 전략을 실험하고 평가하여 최적의 조합을 찾는 것이 중요합니다. 🚀
