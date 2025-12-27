---
layout: post
title: "Elasticsearch Vector 필드와 임베딩 검색: 의미 기반 검색 구현"
date: 2025-12-27
categories: [elasticsearch, search]
tags: [Elasticsearch, Vector, 임베딩, Embedding, 벡터검색, kNN, 의미검색, RAG]
---

# Elasticsearch Vector 필드와 임베딩 검색: 의미 기반 검색 구현

이전 글에서 Redis의 영속성 전략(RDB, AOF)을 다뤘는데, 이번에는 **Elasticsearch의 vector(임베딩) 필드와 벡터 검색**에 대해 정리해보겠습니다.

전통적인 키워드 기반 검색은 "주문 결제"라는 키워드가 정확히 일치해야 검색이 되지만, **벡터 검색(Vector Search)**을 사용하면 "결제 실패", "주문 취소", "결제 오류"처럼 **의미가 비슷한 문장도 검색**할 수 있습니다.

이 글에서는 **Elasticsearch에서 vector 필드를 사용한 임베딩 검색**을 어떻게 구현하는지, 그리고 실제 사용 사례와 주의사항을 정리해보겠습니다.

---

## 1. 임베딩(Embedding)과 벡터 검색이란?

### 임베딩의 개념

**임베딩(Embedding)**은 텍스트, 이미지, 오디오 등의 데이터를 **고정 길이의 실수 벡터로 변환**한 것입니다.

**예시:**
```
텍스트: "주문 결제 실패"
  ↓ (임베딩 모델)
벡터: [0.12, -0.43, 0.87, 0.23, -0.56, ...]  (예: 768차원)
```

**특징:**
- 의미가 비슷한 문장은 **벡터 공간에서도 가까운 거리**를 가짐
- 예: "주문 결제 실패"와 "결제 오류 발생"은 벡터 공간에서 가까움
- 예: "주문 결제 실패"와 "날씨가 좋다"는 벡터 공간에서 멀리 떨어져 있음

### 벡터 검색의 동작 원리

```
1. 문서 인덱싱:
   "주문 결제 실패" → 임베딩 모델 → 벡터 [0.12, -0.43, ...]
   → Elasticsearch에 벡터 저장

2. 검색 쿼리:
   "결제 오류" → 임베딩 모델 → 벡터 [0.15, -0.41, ...]
   → Elasticsearch에서 유사한 벡터 검색

3. 유사도 계산:
   코사인 유사도, L2 거리, dot-product 등으로 유사도 계산
   → 가장 유사한 문서 반환
```

---

## 2. Elasticsearch Vector 필드 타입

### dense_vector 필드

Elasticsearch 7.x부터 **`dense_vector`** 필드 타입을 지원합니다.

**특징:**
- 고정 길이의 실수 벡터 배열을 저장
- 차원(dimension) 수를 미리 정의해야 함
- 예: 768차원, 1536차원 등 (임베딩 모델에 따라 다름)

### 매핑 예시

```json
PUT /product-index
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text"
      },
      "description": {
        "type": "text"
      },
      "title_embedding": {
        "type": "dense_vector",
        "dims": 768,
        "index": true,
        "similarity": "cosine"
      },
      "description_embedding": {
        "type": "dense_vector",
        "dims": 768,
        "index": true,
        "similarity": "cosine"
      }
    }
  }
}
```

**매핑 파라미터 설명:**
- `dims`: 벡터의 차원 수 (예: 768, 1536)
- `index`: `true`로 설정하면 kNN 검색 가능 (Elasticsearch 8.0+)
- `similarity`: 유사도 계산 방식
  - `cosine`: 코사인 유사도 (일반적으로 가장 많이 사용)
  - `l2_norm`: L2 거리 (유클리드 거리)
  - `dot_product`: 내적

---

## 3. 문서 인덱싱: 임베딩 생성 및 저장

### 임베딩 생성 방법

**1) 외부에서 임베딩 생성 후 Elasticsearch에 저장**

```java
// Spring Boot 예시
@Service
public class ProductService {
    
    @Autowired
    private EmbeddingService embeddingService;  // OpenAI, Cohere, HuggingFace 등
    
    @Autowired
    private ElasticsearchRestTemplate esTemplate;
    
    public void indexProduct(Product product) {
        // 1. 텍스트를 임베딩으로 변환
        float[] titleEmbedding = embeddingService.embed(product.getTitle());
        float[] descEmbedding = embeddingService.embed(product.getDescription());
        
        // 2. Elasticsearch에 저장
        Map<String, Object> doc = Map.of(
            "title", product.getTitle(),
            "description", product.getDescription(),
            "title_embedding", titleEmbedding,
            "description_embedding", descEmbedding
        );
        
        esTemplate.save(doc, IndexCoordinates.of("product-index"));
    }
}
```

**2) Elasticsearch Ingest Pipeline 사용 (Elasticsearch 8.0+)**

```json
PUT _ingest/pipeline/embedding-pipeline
{
  "processors": [
    {
      "inference": {
        "model_id": "text-embedding-model",
        "field_map": {
          "title": "text_field"
        },
        "target_field": "title_embedding"
      }
    }
  ]
}

PUT /product-index/_doc/1?pipeline=embedding-pipeline
{
  "title": "주문 결제 실패",
  "description": "결제 처리 중 오류가 발생했습니다"
}
```

### 실제 인덱싱 예시

```json
PUT /product-index/_doc/1
{
  "title": "스마트폰 케이스",
  "description": "아이폰 15 프로용 실리콘 케이스",
  "title_embedding": [0.12, -0.43, 0.87, 0.23, -0.56, ...],
  "description_embedding": [0.15, -0.41, 0.89, 0.25, -0.54, ...]
}
```

---

## 4. 벡터 검색 쿼리

### kNN (k-Nearest Neighbors) 검색

Elasticsearch 8.0+에서는 **`knn` 쿼리**를 사용하여 벡터 검색을 수행합니다.

```json
GET /product-index/_search
{
  "knn": {
    "field": "title_embedding",
    "query_vector": [0.12, -0.43, 0.87, 0.23, -0.56, ...],
    "k": 10,
    "num_candidates": 100
  }
}
```

**파라미터 설명:**
- `field`: 검색할 벡터 필드명
- `query_vector`: 검색 쿼리의 임베딩 벡터
- `k`: 반환할 최상위 문서 수
- `num_candidates`: 후보 문서 수 (k보다 크게 설정, 정확도와 성능의 트레이드오프)

### 하이브리드 검색: 키워드 + 벡터

**키워드 검색과 벡터 검색을 함께 사용**할 수 있습니다.

```json
GET /product-index/_search
{
  "query": {
    "match": {
      "title": "스마트폰"
    }
  },
  "knn": {
    "field": "title_embedding",
    "query_vector": [0.12, -0.43, 0.87, ...],
    "k": 10,
    "num_candidates": 100
  },
  "size": 20
}
```

**장점:**
- 키워드 검색: 정확한 키워드 매칭
- 벡터 검색: 의미 기반 유사도 검색
- 두 결과를 **Reranking**하여 최종 결과 반환

---

## 5. 임베딩 모델 선택

### 모델 종류

**1) OpenAI Embeddings**
- `text-embedding-ada-002`: 1536차원
- `text-embedding-3-small`: 1536차원
- `text-embedding-3-large`: 3072차원
- 장점: 높은 품질, API 사용 간편
- 단점: 비용 발생, API 호출 필요

**2) HuggingFace 모델 (로컬 실행)**
- `sentence-transformers/all-MiniLM-L6-v2`: 384차원
- `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2`: 384차원
- 장점: 무료, 로컬 실행 가능
- 단점: 모델 로딩/관리 필요, GPU 권장

**3) Cohere Embeddings**
- `embed-english-v3.0`: 1024차원
- 장점: 높은 품질, 다국어 지원
- 단점: 비용 발생

**4) Elasticsearch Native 모델 (8.0+)**
- Elasticsearch에 모델을 등록하여 Ingest Pipeline에서 사용
- 장점: Elasticsearch와 통합, 실시간 임베딩 생성
- 단점: 모델 관리 필요

### 모델 선택 가이드

**한국어 검색:**
- HuggingFace의 **다국어 모델** 또는 **한국어 특화 모델** 권장
- 예: `jhgan/ko-sroberta-multitask`, `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2`

**영어 검색:**
- OpenAI `text-embedding-3-small` 또는 HuggingFace `all-MiniLM-L6-v2`

**차원 수 고려:**
- 차원이 높을수록 정확도는 높지만, **인덱스 크기와 검색 속도에 영향**
- 일반적으로 384~1536차원이 실용적

---

## 6. 실전 구현 예시: Spring Boot + Elasticsearch

### 의존성 추가

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
    </dependency>
    <!-- OpenAI Java SDK (예시) -->
    <dependency>
        <groupId>com.theokanning.openai-gpt3-java</groupId>
        <artifactId>service</artifactId>
        <version>0.18.2</version>
    </dependency>
</dependencies>
```

### Entity 정의

```java
@Document(indexName = "product-index")
public class Product {
    
    @Id
    private String id;
    
    @Field(type = FieldType.Text)
    private String title;
    
    @Field(type = FieldType.Text)
    private String description;
    
    @Field(type = FieldType.Dense_Vector, dims = 768)
    private float[] titleEmbedding;
    
    @Field(type = FieldType.Dense_Vector, dims = 768)
    private float[] descriptionEmbedding;
    
    // getter, setter
}
```

### 임베딩 생성 및 인덱싱

```java
@Service
public class ProductIndexingService {
    
    @Autowired
    private EmbeddingService embeddingService;
    
    @Autowired
    private ElasticsearchRestTemplate esTemplate;
    
    public void indexProduct(Product product) {
        // 임베딩 생성
        product.setTitleEmbedding(
            embeddingService.embed(product.getTitle())
        );
        product.setDescriptionEmbedding(
            embeddingService.embed(product.getDescription())
        );
        
        // Elasticsearch에 저장
        esTemplate.save(product);
    }
}
```

### 벡터 검색

```java
@Service
public class ProductSearchService {
    
    @Autowired
    private ElasticsearchRestTemplate esTemplate;
    
    @Autowired
    private EmbeddingService embeddingService;
    
    public List<Product> searchByVector(String queryText) {
        // 1. 쿼리 텍스트를 임베딩으로 변환
        float[] queryVector = embeddingService.embed(queryText);
        
        // 2. kNN 검색 쿼리 생성
        NativeSearchQuery searchQuery = new NativeSearchQueryBuilder()
            .withKnnQuery(
                KnnQuery.of(k -> k
                    .field("title_embedding")
                    .queryVector(queryVector)
                    .k(10)
                    .numCandidates(100)
                )
            )
            .build();
        
        // 3. 검색 실행
        SearchHits<Product> searchHits = esTemplate.search(
            searchQuery, 
            Product.class
        );
        
        return searchHits.stream()
            .map(SearchHit::getContent)
            .toList();
    }
}
```

---

## 7. 성능 최적화 및 주의사항

### 인덱스 성능

**벡터 필드 인덱싱:**
- `index: true`로 설정하면 **HNSW(Hierarchical Navigable Small World) 알고리즘** 사용
- 메모리 사용량이 증가하지만, 검색 속도가 크게 향상됨

**차원 수와 성능:**
- 차원이 높을수록 인덱스 크기와 검색 시간이 증가
- 768차원 기준으로 문서 100만 개 → 약 수 GB 인덱스 크기

### 메모리 관리

**벡터 필드는 메모리에 상당한 부담:**
- 예: 768차원 × 4바이트(float) × 100만 문서 = 약 3GB
- Elasticsearch 힙 메모리 설정을 충분히 할당해야 함

### 검색 정확도 vs 성능

**`num_candidates` 파라미터:**
- 작게 설정: 빠르지만 정확도 낮음
- 크게 설정: 느리지만 정확도 높음
- 일반적으로 `k`의 10~20배 권장

---

## 8. 사용 사례

### 1. 의미 기반 검색

**전통적인 키워드 검색:**
```
검색어: "주문 결제 실패"
결과: "주문 결제 실패"만 검색됨
```

**벡터 검색:**
```
검색어: "주문 결제 실패"
결과: 
  - "주문 결제 실패" (유사도: 0.95)
  - "결제 오류 발생" (유사도: 0.89)
  - "주문 취소 처리" (유사도: 0.82)
  - "결제 시스템 장애" (유사도: 0.78)
```

### 2. RAG (Retrieval-Augmented Generation)

**RAG 아키텍처:**
```
사용자 질문
  ↓
임베딩 변환
  ↓
Elasticsearch 벡터 검색
  ↓
관련 문서 검색
  ↓
LLM에 컨텍스트 제공
  ↓
최종 답변 생성
```

**구현 예시:**
```java
public String answerQuestion(String question) {
    // 1. 질문을 임베딩으로 변환
    float[] queryVector = embeddingService.embed(question);
    
    // 2. Elasticsearch에서 관련 문서 검색
    List<Document> relevantDocs = searchByVector(queryVector);
    
    // 3. 검색된 문서를 컨텍스트로 LLM에 전달
    String context = relevantDocs.stream()
        .map(Document::getContent)
        .collect(Collectors.joining("\n"));
    
    // 4. LLM이 컨텍스트 기반으로 답변 생성
    return llmService.generateAnswer(question, context);
}
```

### 3. 추천 시스템

- 상품 설명, 리뷰 텍스트를 임베딩으로 변환
- 사용자가 본 상품의 임베딩과 유사한 상품 검색
- 키워드 기반 추천보다 **의미 기반 추천**이 더 정확할 수 있음

---

## 9. 하이브리드 검색: 키워드 + 벡터

### RRF (Reciprocal Rank Fusion)

Elasticsearch 8.8+에서는 **RRF**를 사용하여 키워드 검색과 벡터 검색 결과를 결합할 수 있습니다.

```json
GET /product-index/_search
{
  "query": {
    "match": {
      "title": "스마트폰"
    }
  },
  "knn": {
    "field": "title_embedding",
    "query_vector": [0.12, -0.43, ...],
    "k": 10
  },
  "rank": {
    "rrf": {
      "window_size": 20,
      "rank_constant": 20
    }
  }
}
```

**장점:**
- 키워드 검색의 **정확성** + 벡터 검색의 **의미 기반 유연성**
- 두 검색 결과를 자연스럽게 결합

---

## 10. 실전 고려사항

### 임베딩 모델 버전 관리

- 임베딩 모델이 변경되면 **기존 벡터를 모두 재생성**해야 함
- 모델 버전을 필드명에 포함: `title_embedding_v2`, `title_embedding_v3`

### 비용 관리

**OpenAI API 사용 시:**
- 문서 수가 많을수록 임베딩 생성 비용 증가
- 예: 100만 문서 × $0.0001/1K tokens = $100+
- **캐싱 전략** 필요: 동일 텍스트는 재생성하지 않음

### 모니터링

- 벡터 검색 쿼리 응답 시간
- 인덱스 크기, 메모리 사용량
- 검색 정확도 (정성적 평가 또는 A/B 테스트)

---

## 마무리

**핵심 포인트:**

- **Elasticsearch의 `dense_vector` 필드**를 사용하면 임베딩 기반 의미 검색을 구현할 수 있습니다.
- **kNN 검색**으로 유사한 벡터를 찾아 의미가 비슷한 문서를 검색할 수 있습니다.
- **키워드 검색과 벡터 검색을 결합**하면 정확성과 유연성을 모두 확보할 수 있습니다.
- 임베딩 모델 선택, 차원 수, 인덱스 설정은 **성능과 비용, 정확도**를 고려하여 결정해야 합니다.

벡터 검색은 전통적인 키워드 검색의 한계를 넘어 **의미 기반 검색**을 가능하게 하며, RAG, 추천 시스템 등 다양한 AI 애플리케이션의 기반이 됩니다. 🚀

다음 글에서는 이러한 벡터 검색을 활용한 **RAG(Retrieval-Augmented Generation) 시스템 설계**와 실제 도메인(음식 레시피)에 적용할 때의 고민을 정리해보겠습니다.

