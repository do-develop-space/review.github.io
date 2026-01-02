---
layout: post
title: "RAG에서 음식량 분류: 1인분, 2인분 서빙 사이즈 기반 검색과 스케일링"
date: 2026-01-02
categories: [ai, search, architecture]
tags: [RAG, 서빙사이즈, 음식량, 레시피스케일링, Elasticsearch, 필터링, 메타데이터]
---

# RAG에서 음식량 분류: 1인분, 2인분 서빙 사이즈 기반 검색과 스케일링

이전 글에서 음식 레시피 RAG 시스템의 기본 설계를 다뤘습니다. 이번 글에서는 **서빙 사이즈(1인분, 2인분 등)를 기반으로 한 레시피 분류와 검색**을 구현하는 방법을 정리해보겠습니다.

사용자가 "1인분 레시피 알려줘" 또는 "2인분으로 만들고 싶어"라고 질문할 때, RAG 시스템이 어떻게 이를 처리할 수 있는지 살펴보겠습니다.

---

## 1. 서빙 사이즈 분류의 필요성

### 1.1 사용자 시나리오

**일반적인 질문 유형:**
1. **"1인분 김치찌개 레시피 알려줘"**
2. **"2인분으로 만들고 싶은데 재료 양이 얼마나 필요해?"**
3. **"이 레시피를 1인분으로 줄일 수 있어?"**
4. **"4인분 레시피를 2인분으로 만들려면?"**

**문제점:**
- 레시피 데이터베이스에 다양한 서빙 사이즈의 레시피가 혼재
- 사용자가 원하는 서빙 사이즈와 일치하지 않을 수 있음
- 레시피 스케일링(재료 양 조정)이 필요할 수 있음

### 1.2 RAG에서의 고려사항

**기존 RAG의 한계:**
- 벡터 검색만으로는 서빙 사이즈를 정확히 필터링하기 어려움
- "1인분"이라는 키워드가 임베딩에 포함되어도 정확도가 낮을 수 있음
- 메타데이터 필터링과 벡터 검색의 조합이 필요

---

## 2. 데이터 구조 설계

### 2.1 서빙 사이즈 메타데이터 추가

**기본 레시피 구조:**

```java
@Entity
public class Recipe {
    @Id
    private String recipeId;
    
    private String title;
    private String description;
    
    // 서빙 사이즈 정보
    private Integer servings;  // 기본 서빙 사이즈 (예: 2인분)
    private ServingSize servingSize;  // ENUM: ONE, TWO, THREE, FOUR, MANY
    
    // 재료 목록
    @OneToMany(cascade = CascadeType.ALL)
    private List<Ingredient> ingredients;  // servings 기준으로 정규화된 양
    
    // 조리법
    private String instructions;
    
    // 기타 메타데이터
    private String cuisine;
    private Integer cookingTime;
    private Integer difficulty;
}
```

**서빙 사이즈 Enum:**

```java
public enum ServingSize {
    ONE(1, "1인분"),
    TWO(2, "2인분"),
    THREE(3, "3인분"),
    FOUR(4, "4인분"),
    MANY(5, "5인분 이상");
    
    private final int value;
    private final String label;
    
    ServingSize(int value, String label) {
        this.value = value;
        this.label = label;
    }
    
    public static ServingSize fromServings(Integer servings) {
        if (servings == null) return TWO;  // 기본값
        
        return switch (servings) {
            case 1 -> ONE;
            case 2 -> TWO;
            case 3 -> THREE;
            case 4 -> FOUR;
            default -> MANY;
        };
    }
}
```

**재료 구조:**

```java
@Entity
public class Ingredient {
    @Id
    private String ingredientId;
    
    private String name;  // 재료명 (예: "돼지고기")
    private Double amount;  // 양 (예: 200.0)
    private String unit;  // 단위 (예: "g", "ml", "개")
    
    // 이 재료가 기준이 되는 서빙 사이즈
    private Integer baseServings;  // 예: 2인분 기준
    
    @ManyToOne
    private Recipe recipe;
}
```

### 2.2 Elasticsearch 인덱스 설계

**인덱스 매핑:**

```json
{
  "mappings": {
    "properties": {
      "recipe_id": { "type": "keyword" },
      "title": { 
        "type": "text",
        "analyzer": "standard"
      },
      "servings": { 
        "type": "integer" 
      },
      "serving_size": { 
        "type": "keyword"  // "ONE", "TWO", "THREE", "FOUR", "MANY"
      },
      "ingredients": {
        "type": "nested",
        "properties": {
          "name": { "type": "text" },
          "amount": { "type": "float" },
          "unit": { "type": "keyword" },
          "base_servings": { "type": "integer" }
        }
      },
      "instructions": { 
        "type": "text" 
      },
      "embedding": {
        "type": "dense_vector",
        "dims": 384,
        "index": true,
        "similarity": "cosine"
      },
      "metadata": {
        "properties": {
          "cuisine": { "type": "keyword" },
          "cooking_time": { "type": "integer" },
          "difficulty": { "type": "integer" }
        }
      }
    }
  }
}
```

---

## 3. 서빙 사이즈 기반 검색 전략

### 3.1 의도 분류 (Intent Classification)

**사용자 질문에서 서빙 사이즈 추출:**

```java
@Service
public class ServingSizeIntentClassifier {
    
    public ServingSizeIntent classify(String userQuery) {
        // 1. 명시적 서빙 사이즈 추출
        ServingSize explicitSize = extractExplicitServingSize(userQuery);
        if (explicitSize != null) {
            return ServingSizeIntent.builder()
                .intentType(ServingIntentType.EXPLICIT_FILTER)
                .targetServings(explicitSize)
                .build();
        }
        
        // 2. 스케일링 의도 추출 ("2인분으로 만들고 싶어")
        ServingSizeScale scale = extractScalingIntent(userQuery);
        if (scale != null) {
            return ServingSizeIntent.builder()
                .intentType(ServingIntentType.SCALE)
                .sourceServings(scale.getSource())
                .targetServings(scale.getTarget())
                .build();
        }
        
        // 3. 기본값 (서빙 사이즈 필터링 없음)
        return ServingSizeIntent.builder()
            .intentType(ServingIntentType.NO_FILTER)
            .build();
    }
    
    private ServingSize extractExplicitServingSize(String query) {
        // 정규표현식으로 "1인분", "2인분" 등 추출
        Pattern pattern = Pattern.compile("(\\d+)인분");
        Matcher matcher = pattern.matcher(query);
        
        if (matcher.find()) {
            int servings = Integer.parseInt(matcher.group(1));
            return ServingSize.fromServings(servings);
        }
        
        // 키워드 기반 추출
        if (query.contains("1인분") || query.contains("한 사람")) {
            return ServingSize.ONE;
        } else if (query.contains("2인분") || query.contains("두 사람")) {
            return ServingSize.TWO;
        } else if (query.contains("3인분") || query.contains("세 사람")) {
            return ServingSize.THREE;
        } else if (query.contains("4인분") || query.contains("네 사람")) {
            return ServingSize.FOUR;
        }
        
        return null;
    }
    
    private ServingSizeScale extractScalingIntent(String query) {
        // "2인분으로 만들고 싶어", "1인분으로 줄이고 싶어" 등
        Pattern scalePattern = Pattern.compile("(\\d+)인분으로\\s*(만들|줄이|늘리)");
        Matcher matcher = scalePattern.matcher(query);
        
        if (matcher.find()) {
            int targetServings = Integer.parseInt(matcher.group(1));
            // 원본 레시피의 서빙 사이즈는 검색 후에 알 수 있음
            // 여기서는 타겟만 추출
            return ServingSizeScale.builder()
                .target(ServingSize.fromServings(targetServings))
                .build();
        }
        
        return null;
    }
}
```

**의도 타입:**

```java
public enum ServingIntentType {
    EXPLICIT_FILTER,  // "1인분 레시피 알려줘" → 서빙 사이즈 필터링
    SCALE,            // "2인분으로 만들고 싶어" → 레시피 스케일링
    NO_FILTER         // 서빙 사이즈 언급 없음
}

@Data
@Builder
public class ServingSizeIntent {
    private ServingIntentType intentType;
    private ServingSize targetServings;
    private ServingSize sourceServings;
}
```

### 3.2 Elasticsearch 쿼리 구성

**서빙 사이즈 필터링 + 벡터 검색:**

```java
@Service
public class RecipeRAGService {
    
    private final ElasticsearchClient esClient;
    private final EmbeddingService embeddingService;
    private final ServingSizeIntentClassifier intentClassifier;
    
    public List<Recipe> searchWithServingSize(String userQuery, int topK) {
        // 1. 의도 분류
        ServingSizeIntent intent = intentClassifier.classify(userQuery);
        
        // 2. 사용자 질문을 임베딩으로 변환
        float[] queryEmbedding = embeddingService.embed(userQuery);
        
        // 3. Elasticsearch 쿼리 구성
        SearchRequest.Builder searchBuilder = new SearchRequest.Builder()
            .index("recipes")
            .size(topK);
        
        BoolQuery.Builder boolQuery = new BoolQuery.Builder();
        
        // 3-1. 서빙 사이즈 필터링 (명시적 필터링)
        if (intent.getIntentType() == ServingIntentType.EXPLICIT_FILTER) {
            boolQuery.filter(f -> f
                .term(t -> t
                    .field("serving_size")
                    .value(intent.getTargetServings().name())
                )
            );
        }
        
        // 3-2. 벡터 검색 (의미 기반)
        boolQuery.must(m -> m
            .knn(k -> k
                .field("embedding")
                .queryVector(queryEmbedding)
                .k(topK)
            )
        );
        
        // 3-3. 키워드 검색 (제목, 설명)
        if (hasKeywords(userQuery)) {
            boolQuery.should(s -> s
                .match(m -> m
                    .field("title")
                    .query(userQuery)
                    .boost(2.0f)
                )
            ).should(s -> s
                .match(m -> m
                    .field("description")
                    .query(userQuery)
                    .boost(1.0f)
                )
            );
        }
        
        searchBuilder.query(boolQuery.build()._toQuery());
        
        // 4. 검색 실행
        SearchResponse<RecipeDocument> response = esClient.search(
            searchBuilder.build(), 
            RecipeDocument.class
        );
        
        // 5. 결과 처리
        List<Recipe> recipes = response.hits().hits().stream()
            .map(hit -> convertToRecipe(hit.source()))
            .collect(Collectors.toList());
        
        // 6. 스케일링 필요 시 재료 양 조정
        if (intent.getIntentType() == ServingIntentType.SCALE) {
            recipes = scaleRecipes(recipes, intent.getTargetServings());
        }
        
        return recipes;
    }
}
```

### 3.3 서빙 사이즈 범위 검색

**유연한 필터링 (1인분 ±1 범위):**

```java
public List<Recipe> searchWithServingSizeRange(String userQuery, ServingSize targetSize) {
    BoolQuery.Builder boolQuery = new BoolQuery.Builder();
    
    // 서빙 사이즈 범위 필터
    // 예: 1인분을 원하면 1인분 또는 2인분도 포함
    List<String> allowedSizes = new ArrayList<>();
    allowedSizes.add(targetSize.name());
    
    // 인접한 서빙 사이즈도 포함
    if (targetSize == ServingSize.ONE) {
        allowedSizes.add(ServingSize.TWO.name());
    } else if (targetSize == ServingSize.TWO) {
        allowedSizes.add(ServingSize.ONE.name());
        allowedSizes.add(ServingSize.THREE.name());
    } else if (targetSize == ServingSize.THREE) {
        allowedSizes.add(ServingSize.TWO.name());
        allowedSizes.add(ServingSize.FOUR.name());
    } else if (targetSize == ServingSize.FOUR) {
        allowedSizes.add(ServingSize.THREE.name());
        allowedSizes.add(ServingSize.MANY.name());
    }
    
    boolQuery.filter(f -> f
        .terms(t -> t
            .field("serving_size")
            .terms(allowedSizes.stream()
                .map(s -> FieldValue.of(s))
                .collect(Collectors.toList())
            )
        )
    );
    
    // 벡터 검색 추가
    // ...
}
```

---

## 4. 레시피 스케일링 (Recipe Scaling)

### 4.1 재료 양 계산

**스케일링 공식:**

```
새로운 재료 양 = (원본 재료 양 / 원본 서빙 사이즈) × 목표 서빙 사이즈
```

**구현:**

```java
@Service
public class RecipeScalingService {
    
    public Recipe scaleRecipe(Recipe originalRecipe, ServingSize targetServings) {
        if (originalRecipe.getServings() == targetServings.getValue()) {
            return originalRecipe;  // 스케일링 불필요
        }
        
        double scaleFactor = (double) targetServings.getValue() / originalRecipe.getServings();
        
        Recipe scaledRecipe = new Recipe();
        scaledRecipe.setRecipeId(originalRecipe.getRecipeId() + "_scaled_" + targetServings.getValue());
        scaledRecipe.setTitle(originalRecipe.getTitle() + " (" + targetServings.getLabel() + ")");
        scaledRecipe.setServings(targetServings.getValue());
        scaledRecipe.setServingSize(targetServings);
        
        // 재료 양 스케일링
        List<Ingredient> scaledIngredients = originalRecipe.getIngredients().stream()
            .map(ingredient -> scaleIngredient(ingredient, scaleFactor, originalRecipe.getServings()))
            .collect(Collectors.toList());
        
        scaledRecipe.setIngredients(scaledIngredients);
        
        // 조리법은 그대로 (시간, 온도 등은 서빙 사이즈와 무관)
        scaledRecipe.setInstructions(originalRecipe.getInstructions());
        
        return scaledRecipe;
    }
    
    private Ingredient scaleIngredient(Ingredient original, double scaleFactor, int baseServings) {
        Ingredient scaled = new Ingredient();
        scaled.setName(original.getName());
        scaled.setUnit(original.getUnit());
        scaled.setBaseServings(baseServings);
        
        // 재료 양 계산
        double newAmount = original.getAmount() * scaleFactor;
        
        // 단위에 따른 반올림 처리
        if (original.getUnit().equals("개") || original.getUnit().equals("송이")) {
            // 정수 단위는 반올림
            scaled.setAmount((double) Math.round(newAmount));
        } else if (original.getUnit().equals("g") || original.getUnit().equals("ml")) {
            // 소수점 첫째 자리까지
            scaled.setAmount(Math.round(newAmount * 10.0) / 10.0);
        } else {
            scaled.setAmount(newAmount);
        }
        
        return scaled;
    }
    
    public List<Recipe> scaleRecipes(List<Recipe> recipes, ServingSize targetServings) {
        return recipes.stream()
            .map(recipe -> scaleRecipe(recipe, targetServings))
            .collect(Collectors.toList());
    }
}
```

### 4.2 스케일링 시 주의사항

**문제점:**

1. **소수점 처리:**
   - "계란 1.5개" → 반올림 또는 올림 처리 필요

2. **단위 변환:**
   - "물 500ml" → "물 250ml" (1인분으로 줄일 때)
   - 큰 단위로 변환: "물 1000ml" → "물 1L"

3. **조리 시간 조정:**
   - 서빙 사이즈가 크면 조리 시간이 약간 늘어날 수 있음
   - 하지만 일반적으로는 조리 시간은 그대로 유지

**개선된 스케일링:**

```java
private Ingredient scaleIngredient(Ingredient original, double scaleFactor, int baseServings) {
    Ingredient scaled = new Ingredient();
    scaled.setName(original.getName());
    scaled.setBaseServings(baseServings);
    
    double newAmount = original.getAmount() * scaleFactor;
    
    // 단위별 처리
    switch (original.getUnit()) {
        case "개", "송이", "줄기":
            // 정수 단위: 반올림 (0.5 이상은 올림)
            scaled.setAmount((double) Math.round(newAmount));
            scaled.setUnit(original.getUnit());
            break;
            
        case "g", "ml":
            // 소수점 첫째 자리까지
            scaled.setAmount(Math.round(newAmount * 10.0) / 10.0);
            scaled.setUnit(original.getUnit());
            
            // 큰 단위로 변환 (1000 이상이면)
            if (scaled.getAmount() >= 1000 && original.getUnit().equals("ml")) {
                scaled.setAmount(scaled.getAmount() / 1000.0);
                scaled.setUnit("L");
            } else if (scaled.getAmount() >= 1000 && original.getUnit().equals("g")) {
                scaled.setAmount(scaled.getAmount() / 1000.0);
                scaled.setUnit("kg");
            }
            break;
            
        case "큰술", "작은술":
            // 소수점 첫째 자리까지
            scaled.setAmount(Math.round(newAmount * 10.0) / 10.0);
            scaled.setUnit(original.getUnit());
            break;
            
        default:
            scaled.setAmount(newAmount);
            scaled.setUnit(original.getUnit());
    }
    
    return scaled;
}
```

---

## 5. RAG 프롬프트에 서빙 사이즈 정보 포함

### 5.1 프롬프트 구성

**서빙 사이즈 정보를 포함한 프롬프트:**

```java
@Service
public class RecipePromptBuilder {
    
    public String buildPrompt(List<Recipe> recipes, String userQuery, ServingSizeIntent intent) {
        StringBuilder prompt = new StringBuilder();
        
        prompt.append("다음 레시피 정보를 참고하여 질문에 답하세요.\n\n");
        
        for (Recipe recipe : recipes) {
            prompt.append(String.format("""
                레시피: %s
                서빙 사이즈: %s
                재료:
                %s
                조리법: %s
                ---
                """,
                recipe.getTitle(),
                recipe.getServingSize().getLabel(),
                formatIngredients(recipe.getIngredients()),
                recipe.getInstructions()
            ));
        }
        
        // 스케일링 의도가 있으면 추가 안내
        if (intent.getIntentType() == ServingIntentType.SCALE) {
            prompt.append(String.format(
                "\n참고: 위 레시피는 %s 기준입니다. %s로 만들고 싶다면 재료 양을 조정해야 합니다.\n",
                recipes.get(0).getServingSize().getLabel(),
                intent.getTargetServings().getLabel()
            ));
        }
        
        prompt.append(String.format("\n질문: %s\n답변:", userQuery));
        
        return prompt.toString();
    }
    
    private String formatIngredients(List<Ingredient> ingredients) {
        return ingredients.stream()
            .map(ing -> String.format("- %s: %.1f %s", 
                ing.getName(), 
                ing.getAmount(), 
                ing.getUnit()))
            .collect(Collectors.joining("\n"));
    }
}
```

### 5.2 LLM에게 스케일링 요청

**Few-shot 예시 포함:**

```java
public String buildScalingPrompt(Recipe recipe, ServingSize targetServings) {
    return String.format("""
        다음 레시피를 %s로 조정해주세요.
        
        원본 레시피 (%s):
        %s
        
        재료:
        %s
        
        조리법:
        %s
        
        ---
        
        예시:
        원본 (2인분): 돼지고기 200g, 김치 300g
        조정 (1인분): 돼지고기 100g, 김치 150g
        
        ---
        
        위 레시피를 %s로 조정한 재료 목록을 알려주세요.
        """,
        targetServings.getLabel(),
        recipe.getServingSize().getLabel(),
        recipe.getTitle(),
        formatIngredients(recipe.getIngredients()),
        recipe.getInstructions(),
        targetServings.getLabel()
    );
}
```

---

## 6. 실전 구현 예시

### 6.1 전체 RAG 파이프라인

```java
@Service
@Slf4j
public class ServingSizeAwareRecipeRAGService {
    
    private final ElasticsearchClient esClient;
    private final EmbeddingService embeddingService;
    private final ServingSizeIntentClassifier intentClassifier;
    private final RecipeScalingService scalingService;
    private final RecipePromptBuilder promptBuilder;
    private final LLMService llmService;
    
    public String answer(String userQuery) {
        // 1. 의도 분류
        ServingSizeIntent intent = intentClassifier.classify(userQuery);
        log.info("Serving size intent: {}", intent);
        
        // 2. 사용자 질문을 임베딩으로 변환
        float[] queryEmbedding = embeddingService.embed(userQuery);
        
        // 3. Elasticsearch 검색 (서빙 사이즈 필터링 포함)
        List<Recipe> recipes = searchWithServingSize(userQuery, intent, 5);
        
        if (recipes.isEmpty()) {
            return "해당 서빙 사이즈의 레시피를 찾을 수 없습니다.";
        }
        
        // 4. 스케일링 필요 시 재료 양 조정
        if (intent.getIntentType() == ServingIntentType.SCALE) {
            recipes = scalingService.scaleRecipes(recipes, intent.getTargetServings());
        }
        
        // 5. 프롬프트 구성
        String prompt = promptBuilder.buildPrompt(recipes, userQuery, intent);
        
        // 6. LLM 호출
        String answer = llmService.generate(prompt);
        
        return answer;
    }
    
    private List<Recipe> searchWithServingSize(String query, ServingSizeIntent intent, int topK) {
        float[] queryEmbedding = embeddingService.embed(query);
        
        BoolQuery.Builder boolQuery = new BoolQuery.Builder();
        
        // 서빙 사이즈 필터링
        if (intent.getIntentType() == ServingIntentType.EXPLICIT_FILTER) {
            boolQuery.filter(f -> f
                .term(t -> t
                    .field("serving_size")
                    .value(intent.getTargetServings().name())
                )
            );
        }
        
        // 벡터 검색
        boolQuery.must(m -> m
            .knn(k -> k
                .field("embedding")
                .queryVector(queryEmbedding)
                .k(topK)
            )
        );
        
        // 키워드 검색
        boolQuery.should(s -> s
            .match(m -> m
                .field("title")
                .query(query)
                .boost(2.0f)
            )
        );
        
        SearchRequest request = SearchRequest.of(s -> s
            .index("recipes")
            .query(boolQuery.build()._toQuery())
            .size(topK)
        );
        
        SearchResponse<RecipeDocument> response = esClient.search(request, RecipeDocument.class);
        
        return response.hits().hits().stream()
            .map(hit -> convertToRecipe(hit.source()))
            .collect(Collectors.toList());
    }
}
```

### 6.2 사용 예시

**시나리오 1: "1인분 김치찌개 레시피 알려줘"**

```java
String query = "1인분 김치찌개 레시피 알려줘";
String answer = ragService.answer(query);

// 결과:
// - ServingSizeIntent: EXPLICIT_FILTER, targetServings=ONE
// - Elasticsearch에서 serving_size="ONE" 필터링
// - 벡터 검색으로 "김치찌개" 관련 레시피 검색
// - 1인분 레시피 반환
```

**시나리오 2: "2인분으로 만들고 싶은데 재료 양이 얼마나 필요해?"**

```java
String query = "2인분으로 만들고 싶은데 재료 양이 얼마나 필요해?";
String answer = ragService.answer(query);

// 결과:
// - ServingSizeIntent: SCALE, targetServings=TWO
// - Elasticsearch에서 레시피 검색 (서빙 사이즈 필터링 없음)
// - 검색된 레시피를 2인분으로 스케일링
// - LLM이 스케일링된 재료 양을 설명
```

**시나리오 3: "이 레시피를 1인분으로 줄일 수 있어?"**

```java
String query = "이 레시피를 1인분으로 줄일 수 있어?";
String answer = ragService.answer(query);

// 결과:
// - ServingSizeIntent: SCALE, targetServings=ONE
// - 원본 레시피의 서빙 사이즈 확인 (예: 4인분)
// - 4인분 → 1인분으로 스케일링 (재료 양 1/4로 조정)
// - LLM이 스케일링된 재료 목록 제공
```

### 6.3 RAG 필요성 검토: GPT-4만으로 충분한가?

각 시나리오별로 **RAG 없이 GPT-4만으로 충분한지** 검토해보겠습니다.

#### 시나리오 1: "1인분 김치찌개 레시피 알려줘"

**GPT-4만 사용할 경우:**

```java
String query = "1인분 김치찌개 레시피 알려줘";
String answer = gpt4Service.generate(query);

// GPT-4 응답:
// "1인분 김치찌개 레시피입니다.
//  재료: 돼지고기 100g, 김치 150g, 물 200ml..."
```

**평가:**

| 항목 | GPT-4만 사용 | RAG 사용 |
|------|-------------|----------|
| **정확도** | ⚠️ 일반적인 레시피 제공 (정확하지만 일반적) | ✅ 실제 DB의 1인분 레시피 제공 |
| **신뢰성** | ❌ 출처 불명확, 환각 가능성 | ✅ 실제 레시피 DB 기반, 출처 명확 |
| **맞춤성** | ❌ 서비스 특화 레시피 불가 | ✅ 서비스의 실제 레시피 제공 |
| **비용** | ✅ 낮음 (단일 LLM 호출) | ⚠️ 중간 (검색 + LLM 호출) |
| **지연 시간** | ✅ 빠름 (~2초) | ⚠️ 느림 (~3-5초) |

**결론:**
- **일반적인 요리 질문**: GPT-4만으로 충분 ✅
- **특정 서비스의 레시피**: RAG 필수 ✅
  - 예: "우리 앱의 1인분 김치찌개 레시피"
  - 서비스에 등록된 실제 레시피 DB에서 검색 필요
- **서비스 특화 레시피**: RAG 필수 ✅
  - 예: "우리 식당의 시그니처 레시피"
  - 일반적인 레시피가 아닌 특정 서비스의 고유 레시피

#### 시나리오 2: "2인분으로 만들고 싶은데 재료 양이 얼마나 필요해?"

**GPT-4만 사용할 경우:**

```java
String query = "2인분으로 만들고 싶은데 재료 양이 얼마나 필요해?";
String answer = gpt4Service.generate(query);

// 문제점:
// 1. 어떤 레시피를 기준으로 할지 모름 (컨텍스트 없음)
// 2. 사용자가 이전에 본 레시피를 참조해야 함
// 3. 일반적인 레시피로 답변 (서비스 특화 불가)
```

**평가:**

| 항목 | GPT-4만 사용 | RAG 사용 |
|------|-------------|----------|
| **컨텍스트 이해** | ❌ 이전 레시피 정보 없음 | ✅ 검색된 레시피 기반 스케일링 |
| **정확도** | ⚠️ 일반적인 답변만 가능 | ✅ 실제 레시피 기반 정확한 계산 |
| **스케일링** | ⚠️ 수동 계산 필요 | ✅ 자동 스케일링 + LLM 설명 |
| **사용자 경험** | ❌ 컨텍스트 부족으로 답변 품질 저하 | ✅ 자연스러운 대화 흐름 |

**결론:**
- **컨텍스트가 필요한 질문**: RAG 필수 ✅
- **이전 대화 맥락 유지**: RAG 필수 ✅
- **서비스 특화 레시피 스케일링**: RAG 필수 ✅

#### 시나리오 3: "이 레시피를 1인분으로 줄일 수 있어?"

**GPT-4만 사용할 경우:**

```java
String query = "이 레시피를 1인분으로 줄일 수 있어?";
String answer = gpt4Service.generate(query);

// 문제점:
// 1. "이 레시피"가 무엇인지 모름 (컨텍스트 없음)
// 2. 사용자가 현재 보고 있는 레시피 정보 필요
// 3. 대화 세션 관리 필요
```

**평가:**

| 항목 | GPT-4만 사용 | RAG 사용 |
|------|-------------|----------|
| **레시피 식별** | ❌ "이 레시피" 참조 불가 | ✅ 검색된 레시피 또는 세션 기반 |
| **정확한 스케일링** | ⚠️ 일반적인 계산만 가능 | ✅ 실제 레시피 기반 정확한 계산 |
| **대화 맥락** | ❌ 세션 관리 어려움 | ✅ 검색 결과로 맥락 유지 |

**결론:**
- **대화형 질문**: RAG 필수 ✅
- **레시피 참조 질문**: RAG 필수 ✅
- **컨텍스트 기반 답변**: RAG 필수 ✅

### 6.4 하이브리드 접근: RAG + GPT-4 조합

**최적의 전략: 질문 유형에 따라 선택**

```java
@Service
public class HybridRecipeService {
    
    private final GPT4Service gpt4Service;
    private final RecipeRAGService ragService;
    private final IntentClassifier intentClassifier;
    
    public String answer(String userQuery, ConversationContext context) {
        // 1. 의도 분류
        RecipeIntent intent = intentClassifier.classify(userQuery, context);
        
        switch (intent.getType()) {
            case GENERAL_COOKING_QUESTION:
                // 일반적인 요리 질문 → GPT-4 직접 사용
                // 예: "왜 고기를 익히기 전에 소금을 뿌리나요?"
                return gpt4Service.generate(userQuery);
                
            case SPECIFIC_RECIPE_REQUEST:
                // 특정 레시피 요청 → RAG 사용
                // 예: "1인분 김치찌개 레시피 알려줘"
                return ragService.answer(userQuery);
                
            case RECIPE_SCALING:
                // 레시피 스케일링 → RAG 필수 (컨텍스트 필요)
                // 예: "2인분으로 만들고 싶어"
                return ragService.answer(userQuery, context);
                
            case RECIPE_REFERENCE:
                // 레시피 참조 질문 → RAG 필수 (세션 기반)
                // 예: "이 레시피를 1인분으로 줄일 수 있어?"
                return ragService.answerWithContext(userQuery, context);
                
            case CREATIVE_RECIPE:
                // 창의적인 레시피 생성 → GPT-4 직접 사용
                // 예: "김치와 계란으로 새로운 요리 만들어줘"
                return gpt4Service.generate(userQuery);
                
            default:
                // 하이브리드: RAG로 검색 + GPT-4로 보강
                List<Recipe> recipes = ragService.search(userQuery, 3);
                if (recipes.isEmpty()) {
                    // 검색 결과 없음 → GPT-4로 일반 답변
                    return gpt4Service.generate(userQuery);
                } else {
                    // 검색 결과 있음 → RAG 사용
                    return ragService.answerWithRecipes(userQuery, recipes);
                }
        }
    }
}
```

### 6.5 RAG 필요성 결정 매트릭스

| 질문 유형 | 예시 | GPT-4만 | RAG 필요 | 이유 |
|----------|------|---------|----------|------|
| **일반 요리 지식** | "왜 고기를 익히기 전에 소금을 뿌리나요?" | ✅ 충분 | ❌ 불필요 | 일반 지식 질문 |
| **요리 원리** | "스테이크 익히는 온도는?" | ✅ 충분 | ❌ 불필요 | 일반 지식 질문 |
| **특정 레시피 요청** | "1인분 김치찌개 레시피" | ⚠️ 일반적 답변 | ✅ 필수 | 서비스 특화 필요 |
| **레시피 스케일링** | "2인분으로 만들고 싶어" | ❌ 컨텍스트 없음 | ✅ 필수 | 원본 레시피 필요 |
| **레시피 참조** | "이 레시피를 1인분으로 줄일 수 있어?" | ❌ 참조 불가 | ✅ 필수 | 세션/컨텍스트 필요 |
| **창의적 레시피** | "김치와 계란으로 새로운 요리" | ✅ 충분 | ❌ 불필요 | 창의성 요구 |
| **서비스 특화** | "우리 앱의 시그니처 레시피" | ❌ 모름 | ✅ 필수 | 실제 DB 필요 |

### 6.6 비용 및 성능 고려사항

**비용 비교:**

```
GPT-4만 사용:
- 입력 토큰: ~100 tokens
- 출력 토큰: ~500 tokens
- 비용: ~$0.01 per request

RAG 사용:
- 임베딩 생성: ~$0.0001
- Elasticsearch 검색: ~$0.0001
- LLM 호출: ~$0.01
- 총 비용: ~$0.01 per request (거의 동일)
```

**지연 시간 비교:**

```
GPT-4만 사용:
- LLM 호출: ~2초
- 총 지연: ~2초

RAG 사용:
- 임베딩 생성: ~0.5초
- Elasticsearch 검색: ~0.1초
- LLM 호출: ~2초
- 총 지연: ~2.6초 (약 30% 증가)
```

**결론:**
- **비용**: 거의 동일 (RAG 오버헤드 미미)
- **지연 시간**: RAG가 약간 느림 (30% 증가)
- **정확도/신뢰성**: RAG가 훨씬 우수 (특정 레시피 질문)

---

## 7. 고급 기능

### 7.1 서빙 사이즈 자동 분류

**레시피 데이터에 서빙 사이즈가 없는 경우:**

```java
@Service
public class ServingSizeClassifier {
    
    public ServingSize classifyFromIngredients(List<Ingredient> ingredients) {
        // 재료 양을 기반으로 서빙 사이즈 추정
        // 예: 돼지고기 200g → 2인분 추정
        
        double totalAmount = ingredients.stream()
            .mapToDouble(Ingredient::getAmount)
            .sum();
        
        // 휴리스틱: 주요 재료 양을 기준으로 추정
        // (도메인별로 조정 필요)
        if (totalAmount < 300) {
            return ServingSize.ONE;
        } else if (totalAmount < 600) {
            return ServingSize.TWO;
        } else if (totalAmount < 900) {
            return ServingSize.THREE;
        } else if (totalAmount < 1200) {
            return ServingSize.FOUR;
        } else {
            return ServingSize.MANY;
        }
    }
    
    public ServingSize classifyFromText(String recipeText) {
        // 텍스트에서 서빙 사이즈 키워드 추출
        if (recipeText.contains("1인분") || recipeText.contains("한 사람")) {
            return ServingSize.ONE;
        } else if (recipeText.contains("2인분") || recipeText.contains("두 사람")) {
            return ServingSize.TWO;
        }
        // ...
        
        return ServingSize.TWO;  // 기본값
    }
}
```

### 7.2 서빙 사이즈별 레시피 추천

**사용자 프로필 기반 추천:**

```java
@Service
public class ServingSizeBasedRecommendation {
    
    public List<Recipe> recommendRecipes(Long userId, int topK) {
        // 사용자 프로필에서 선호 서빙 사이즈 조회
        UserProfile profile = userProfileRepository.findByUserId(userId);
        ServingSize preferredSize = profile.getPreferredServingSize();
        
        // 선호 서빙 사이즈 기반 검색
        BoolQuery.Builder boolQuery = new BoolQuery.Builder();
        boolQuery.filter(f -> f
            .term(t -> t
                .field("serving_size")
                .value(preferredSize.name())
            )
        );
        
        // 인기도, 평점 등으로 정렬
        // ...
        
        return searchRecipes(boolQuery.build(), topK);
    }
}
```

---

## 8. 모니터링 및 개선

### 8.1 핵심 메트릭

1. **서빙 사이즈 분류 정확도**
   - 의도 분류 정확도
   - 사용자 피드백 기반 개선

2. **스케일링 품질**
   - 스케일링된 재료 양의 정확도
   - 사용자 만족도

3. **검색 품질**
   - 서빙 사이즈 필터링 후 검색 결과 관련성
   - 검색 결과 다양성

### 8.2 A/B 테스트

- **서빙 사이즈 범위 검색**: 정확한 매칭 vs 범위 검색
- **스케일링 방식**: 자동 스케일링 vs LLM 기반 스케일링
- **프롬프트 형식**: 서빙 사이즈 정보 포함 방식

---

## 마무리

**핵심 포인트:**

- **서빙 사이즈는 메타데이터 필터링과 벡터 검색을 조합하여 처리할 수 있습니다.**
- **의도 분류를 통해 사용자가 원하는 서빙 사이즈를 추출하고, 적절한 필터링 또는 스케일링을 적용합니다.**
- **레시피 스케일링은 재료 양을 비례적으로 조정하되, 단위와 소수점 처리를 고려해야 합니다.**
- **프롬프트에 서빙 사이즈 정보를 포함하여 LLM이 정확한 답변을 생성하도록 유도합니다.**

RAG 시스템에서 서빙 사이즈 분류를 구현하면, 사용자가 원하는 양에 맞는 레시피를 제공할 수 있어 사용자 만족도가 크게 향상됩니다. 특히 "1인분 레시피"나 "2인분으로 만들고 싶어" 같은 요구사항을 자연스럽게 처리할 수 있습니다.

서빙 사이즈 분류는 RAG 시스템의 한 가지 활용 사례일 뿐입니다. Farm-to-table 플랫폼에서는 영양 정보, 알레르기 정보, 계절별 농산물 추천 등 다양한 AI 추천 시스템을 구현할 수 있으며, 이러한 기능들을 통해 사용자에게 더욱 맞춤형 서비스를 제공할 수 있습니다. 🚀

