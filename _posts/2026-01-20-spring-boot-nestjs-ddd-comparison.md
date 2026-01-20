---
layout: post
title: "Spring Boot vs NestJS: DDD 프로젝트 구조 설계 차이점 완전 정리"
date: 2026-01-20
categories: [architecture, ddd, programming]
tags: [SpringBoot, NestJS, DDD, DomainDrivenDesign, 아키텍처, 프로젝트구조, CleanArchitecture, HexagonalArchitecture, 의존성주입, Repository패턴]
---

이전 글에서 JPA, TypeORM, Django ORM의 N+1 문제 해결 방식을 비교했습니다. 이번 글에서는 **Spring Boot와 NestJS에서 DDD(Domain-Driven Design) 프로젝트 구조를 설계하는 방식의 차이점**을 정리해보겠습니다.

두 프레임워크 모두 DDD, Clean Architecture, Hexagonal Architecture 등의 개념을 적용할 수 있지만, 언어 특성, 프레임워크 철학, 생태계에 따라 구조와 설계 스타일에 차이가 생깁니다. 실제 프로젝트에서 선택할 때 도움이 되도록 상세히 비교해보겠습니다.

---

## 1. DDD 기본 개념 (공통)

### 1.1 계층 구조

두 프레임워크 모두 다음과 같은 계층 구조를 따릅니다:

1. **Domain Layer (도메인 계층)**: 비즈니스 핵심 로직, 프레임워크 독립적
2. **Application Layer (애플리케이션 계층)**: 유스케이스 오케스트레이션
3. **Infrastructure Layer (인프라 계층)**: 기술적 구현 (DB, 외부 API 등)
4. **Presentation Layer (프레젠테이션 계층)**: 외부 인터페이스 (REST API, GraphQL 등)

### 1.2 의존성 방향

**의존성 역전 원칙 (DIP)**:
- 도메인 계층은 다른 계층에 의존하지 않음
- 인프라 계층이 도메인 계층의 인터페이스를 구현
- 의존성은 항상 안쪽(도메인)을 향함

---

## 2. Spring Boot DDD 구조

### 2.1 패키지 기반 계층 구조

**실제 프로젝트 구조 예시:**
```
com.barofarm.seller/
├── seller/
│   ├── domain/                    # 도메인 계층
│   │   ├── Seller.java            # 엔티티 (JPA Entity)
│   │   ├── SellerRepository.java  # Repository 인터페이스
│   │   ├── Status.java            # Value Object
│   │   └── validation/
│   │       └── BusinessValidator.java
│   ├── application/               # 애플리케이션 계층
│   │   ├── SellerService.java     # 유스케이스
│   │   └── dto/
│   ├── infrastructure/            # 인프라 계층
│   │   ├── SellerJpaRepository.java        # Spring Data JPA
│   │   ├── SellerRepositoryAdapter.java    # Adapter 패턴
│   │   └── feign/
│   │       └── AuthClient.java
│   └── presentation/             # 프레젠테이션 계층
│       ├── SellerController.java
│       └── dto/
│           ├── SellerApplyRequestDto.java
│           └── SellerInfoDto.java
└── SellerApplication.java
```

**특징:**
- **패키지로 계층 분리**: `domain`, `application`, `infrastructure`, `presentation`
- **도메인 계층은 프레임워크 독립적**: JPA 어노테이션은 있지만 Spring 어노테이션은 없음
- **Adapter 패턴**: Repository 인터페이스와 구현체 분리

### 2.2 도메인 엔티티 (Domain Entity)

**Spring Boot에서는 도메인 엔티티 = JPA Entity:**

```java
@Entity
@Table(name = "seller")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Seller extends BaseEntity {
    
    @Id
    @Column(name = "user_id", columnDefinition = "BINARY(16)")
    private UUID id;
    
    @Column(name = "store_name", nullable = false, length = 50)
    private String storeName;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "seller_status", nullable = false, length = 20)
    private Status status;
    
    // 비즈니스 로직
    public boolean isActive() {
        return this.status == Status.APPROVED;
    }
    
    // 팩토리 메서드
    public static Seller createApproved(
        UUID userId,
        String storeName,
        String businessRegNo,
        String businessOwnerName,
        String settlementBank,
        String settlementAccount
    ) {
        return new Seller(
            userId,
            storeName,
            businessRegNo,
            businessOwnerName,
            settlementBank,
            settlementAccount,
            Status.APPROVED
        );
    }
}
```

**특징:**
- JPA 어노테이션(`@Entity`, `@Column`) 사용
- 도메인 로직(`isActive()`) 포함
- 팩토리 메서드로 객체 생성 제어
- 도메인 모델과 영속성 모델이 통합됨

### 2.3 Repository 패턴

**도메인 계층: 인터페이스만 정의**

```java
// domain/SellerRepository.java
public interface SellerRepository {
    Seller save(Seller seller);
    Optional<Seller> findById(UUID id);
    List<Seller> findByIdIn(List<UUID> userIds);
}
```

**인프라 계층: Adapter 패턴으로 구현**

```java
// infrastructure/SellerRepositoryAdapter.java
@Repository
@RequiredArgsConstructor
public class SellerRepositoryAdapter implements SellerRepository {
    
    private final SellerJpaRepository sellerJpaRepository;
    
    @Override
    public Seller save(Seller seller) {
        return sellerJpaRepository.save(seller);
    }
    
    @Override
    public Optional<Seller> findById(UUID id) {
        return sellerJpaRepository.findById(id);
    }
    
    @Override
    public List<Seller> findByIdIn(List<UUID> userIds) {
        return sellerJpaRepository.findByIdIn(userIds);
    }
}
```

**Spring Data JPA 인터페이스:**

```java
// infrastructure/SellerJpaRepository.java
public interface SellerJpaRepository extends JpaRepository<Seller, UUID> {
    boolean existsByBusinessRegNo(String businessRegNo);
    List<Seller> findByIdIn(List<UUID> ids);
}
```

**특징:**
- 도메인 계층은 인터페이스만 정의 (프레임워크 독립적)
- Adapter가 Spring Data JPA와 연결
- 도메인 계층은 JPA에 직접 의존하지 않음

### 2.4 애플리케이션 서비스

```java
@Service
@RequiredArgsConstructor
public class SellerService {
    
    private final SellerRepository sellerRepository;  // 인터페이스 주입
    private final AuthClient authClient;
    private final BusinessValidator businessValidator;
    
    @Transactional(readOnly = true)
    public SellerInfoDto getASellerByUserId(UUID userId) {
        Seller seller = sellerRepository.findById(userId)
            .orElseThrow(() -> new CustomException(SELLER_NOT_FOUND));
        return SellerInfoDto.from(seller);
    }
    
    @Transactional
    public void applyForSeller(UUID userId, SellerApplyRequestDto requestDto) {
        // 1. 비즈니스 검증
        businessValidator.validate(userId, requestDto.businessRegNo(), 
                                   requestDto.businessOwnerName());
        
        // 2. 도메인 객체 생성
        Seller profile = Seller.createApproved(
            userId,
            requestDto.storeName(),
            requestDto.businessRegNo(),
            requestDto.businessOwnerName(),
            requestDto.settlementBank(),
            requestDto.settlementAccount()
        );
        
        // 3. 저장
        sellerRepository.save(profile);
        
        // 4. 트랜잭션 커밋 후 외부 서비스 호출
        runAfterCommit(() -> callGrantSellerWithRetry(userId));
    }
}
```

**특징:**
- `@Service` 어노테이션으로 Spring Bean 등록
- `@RequiredArgsConstructor`로 생성자 주입
- `@Transactional`로 선언적 트랜잭션 관리
- Repository 인터페이스 주입 (구현체가 아닌)

### 2.5 트랜잭션 관리

```java
@Service
@Transactional  // 클래스 레벨 기본 설정
public class SellerService {
    
    @Transactional(readOnly = true)  // 읽기 전용 오버라이드
    public SellerInfoDto getASellerByUserId(UUID userId) {
        // ...
    }
    
    @Transactional  // 쓰기 트랜잭션
    public void applyForSeller(UUID userId, SellerApplyRequestDto requestDto) {
        // ...
    }
}
```

**특징:**
- `@Transactional`로 선언적 트랜잭션
- AOP 기반으로 자동 관리
- `readOnly = true`로 읽기 최적화
- 트랜잭션 전파, 격리 수준 등 세밀한 제어 가능

### 2.6 의존성 주입

```java
@Service
@RequiredArgsConstructor  // Lombok으로 생성자 자동 생성
public class SellerService {
    private final SellerRepository sellerRepository;
    private final AuthClient authClient;
    private final BusinessValidator businessValidator;
}
```

**특징:**
- 생성자 주입 (권장)
- `@RequiredArgsConstructor`로 보일러플레이트 제거
- 인터페이스 기반 주입 (의존성 역전)

---

## 3. NestJS DDD 구조

### 3.1 모듈 기반 구조

**실제 프로젝트 구조 예시:**
```
src/
├── seller/
│   ├── domain/
│   │   ├── entities/
│   │   │   └── seller.entity.ts          # Domain Entity (순수)
│   │   ├── repositories/
│   │   │   └── seller.repository.interface.ts
│   │   └── services/
│   │       └── seller.domain.service.ts
│   ├── application/
│   │   ├── services/
│   │   │   └── seller.service.ts
│   │   └── dto/
│   ├── infrastructure/
│   │   ├── persistence/
│   │   │   ├── seller.repository.ts      # 구현체
│   │   │   └── seller.entity.ts          # TypeORM Entity
│   │   └── external/
│   └── presentation/
│       ├── seller.controller.ts
│       └── dto/
└── seller.module.ts                       # NestJS 모듈 정의
```

**특징:**
- **모듈 기반 구조**: `@Module` 데코레이터로 모듈 정의
- **도메인 엔티티와 TypeORM 엔티티 분리**: 순수 도메인 모델 유지
- **의존성 주입**: 모듈에서 Provider 등록

### 3.2 도메인 엔티티 (Domain Entity)

**NestJS에서는 도메인 엔티티와 TypeORM 엔티티를 분리:**

```typescript
// domain/entities/seller.entity.ts
export class Seller {
  private constructor(
    private readonly id: string,
    private readonly storeName: string,
    private readonly businessRegNo: string,
    private readonly businessOwnerName: string,
    private readonly settlementBank: string,
    private readonly settlementAccount: string,
    private readonly status: SellerStatus,
  ) {}
  
  // 비즈니스 로직
  isActive(): boolean {
    return this.status === SellerStatus.APPROVED;
  }
  
  // 팩토리 메서드
  static createApproved(
    userId: string,
    storeName: string,
    businessRegNo: string,
    businessOwnerName: string,
    settlementBank: string,
    settlementAccount: string,
  ): Seller {
    return new Seller(
      userId,
      storeName,
      businessRegNo,
      businessOwnerName,
      settlementBank,
      settlementAccount,
      SellerStatus.APPROVED,
    );
  }
  
  // Getters
  getId(): string { return this.id; }
  getStoreName(): string { return this.storeName; }
  getStatus(): SellerStatus { return this.status; }
}
```

**TypeORM Entity (인프라 계층):**

```typescript
// infrastructure/persistence/seller.entity.ts
import { Entity, PrimaryColumn, Column } from 'typeorm';

@Entity('seller')
export class SellerEntity {
  @PrimaryColumn('uuid')
  userId: string;
  
  @Column()
  storeName: string;
  
  @Column()
  businessRegNo: string;
  
  @Column()
  businessOwnerName: string;
  
  @Column()
  settlementBank: string;
  
  @Column()
  settlementAccount: string;
  
  @Column()
  status: string;
}
```

**특징:**
- 도메인 엔티티는 순수 TypeScript 클래스 (프레임워크 독립적)
- TypeORM Entity는 인프라 계층에만 존재
- Repository에서 매핑 로직 필요

### 3.3 Repository 패턴

**도메인 계층: 인터페이스 정의**

```typescript
// domain/repositories/seller.repository.interface.ts
export interface ISellerRepository {
  save(seller: Seller): Promise<Seller>;
  findById(id: string): Promise<Seller | null>;
  findByIdIn(ids: string[]): Promise<Seller[]>;
}
```

**인프라 계층: 구현체 (매핑 포함)**

```typescript
// infrastructure/persistence/seller.repository.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { ISellerRepository } from '../../domain/repositories/seller.repository.interface';
import { Seller } from '../../domain/entities/seller.entity';
import { SellerEntity } from './seller.entity';

@Injectable()
export class SellerRepository implements ISellerRepository {
  constructor(
    @InjectRepository(SellerEntity)
    private readonly sellerRepo: Repository<SellerEntity>,
  ) {}
  
  async save(seller: Seller): Promise<Seller> {
    const entity = this.toEntity(seller);
    const saved = await this.sellerRepo.save(entity);
    return this.toDomain(saved);
  }
  
  async findById(id: string): Promise<Seller | null> {
    const entity = await this.sellerRepo.findOne({ where: { userId: id } });
    return entity ? this.toDomain(entity) : null;
  }
  
  async findByIdIn(ids: string[]): Promise<Seller[]> {
    const entities = await this.sellerRepo.find({
      where: ids.map(id => ({ userId: id })),
    });
    return entities.map(e => this.toDomain(e));
  }
  
  // 매핑 메서드
  private toEntity(domain: Seller): SellerEntity {
    const entity = new SellerEntity();
    entity.userId = domain.getId();
    entity.storeName = domain.getStoreName();
    entity.businessRegNo = domain.getBusinessRegNo();
    entity.businessOwnerName = domain.getBusinessOwnerName();
    entity.settlementBank = domain.getSettlementBank();
    entity.settlementAccount = domain.getSettlementAccount();
    entity.status = domain.getStatus();
    return entity;
  }
  
  private toDomain(entity: SellerEntity): Seller {
    return Seller.createApproved(
      entity.userId,
      entity.storeName,
      entity.businessRegNo,
      entity.businessOwnerName,
      entity.settlementBank,
      entity.settlementAccount,
    );
  }
}
```

**특징:**
- 도메인 엔티티와 TypeORM 엔티티 분리
- `toEntity()`, `toDomain()` 매핑 메서드 필요
- 도메인 순수성 유지 가능

### 3.4 애플리케이션 서비스

```typescript
// application/services/seller.service.ts
import { Injectable, Inject } from '@nestjs/common';
import { ISellerRepository } from '../../domain/repositories/seller.repository.interface';
import { Seller } from '../../domain/entities/seller.entity';
import { SellerInfoDto } from '../dto/seller-info.dto';
import { SellerApplyRequestDto } from '../dto/seller-apply-request.dto';

@Injectable()
export class SellerService {
  constructor(
    @Inject('ISellerRepository')
    private readonly sellerRepository: ISellerRepository,
    private readonly businessValidator: BusinessValidator,
    private readonly authClient: AuthClient,
  ) {}
  
  async getASellerByUserId(userId: string): Promise<SellerInfoDto> {
    const seller = await this.sellerRepository.findById(userId);
    if (!seller) {
      throw new NotFoundException('Seller not found');
    }
    return SellerInfoDto.from(seller);
  }
  
  async applyForSeller(userId: string, dto: SellerApplyRequestDto): Promise<void> {
    // 1. 비즈니스 검증
    await this.businessValidator.validate(
      userId,
      dto.businessRegNo,
      dto.businessOwnerName,
    );
    
    // 2. 도메인 객체 생성
    const seller = Seller.createApproved(
      userId,
      dto.storeName,
      dto.businessRegNo,
      dto.businessOwnerName,
      dto.settlementBank,
      dto.settlementAccount,
    );
    
    // 3. 저장
    await this.sellerRepository.save(seller);
    
    // 4. 외부 서비스 호출 (비동기)
    await this.authClient.grantSeller(userId);
  }
}
```

**특징:**
- `@Injectable()` 데코레이터로 NestJS DI 등록
- `@Inject()`로 인터페이스 주입 (토큰 기반)
- 비동기 처리 (`async/await`)

### 3.5 트랜잭션 관리

**NestJS는 수동 트랜잭션 관리:**

```typescript
import { Injectable } from '@nestjs/common';
import { DataSource } from 'typeorm';

@Injectable()
export class SellerService {
  constructor(
    @InjectDataSource()
    private readonly dataSource: DataSource,
    @Inject('ISellerRepository')
    private readonly sellerRepository: ISellerRepository,
  ) {}
  
  async applyForSeller(userId: string, dto: SellerApplyRequestDto): Promise<void> {
    const queryRunner = this.dataSource.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();
    
    try {
      // 비즈니스 로직
      const seller = Seller.createApproved(/* ... */);
      await this.sellerRepository.save(seller);
      
      await queryRunner.commitTransaction();
    } catch (error) {
      await queryRunner.rollbackTransaction();
      throw error;
    } finally {
      await queryRunner.release();
    }
  }
}
```

**또는 `@Transactional()` 데코레이터 라이브러리 사용:**

```typescript
import { Transactional } from 'typeorm-transactional';

@Injectable()
export class SellerService {
  @Transactional()
  async applyForSeller(userId: string, dto: SellerApplyRequestDto): Promise<void> {
    // 비즈니스 로직
  }
}
```

**특징:**
- 수동 트랜잭션 관리 (QueryRunner)
- 또는 라이브러리 사용 (`typeorm-transactional`)
- Spring보다 명시적 관리 필요

### 3.6 모듈 정의

```typescript
// seller.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { SellerController } from './presentation/seller.controller';
import { SellerService } from './application/services/seller.service';
import { SellerRepository } from './infrastructure/persistence/seller.repository';
import { SellerEntity } from './infrastructure/persistence/seller.entity';

@Module({
  imports: [
    TypeOrmModule.forFeature([SellerEntity]),  // TypeORM 모듈
  ],
  providers: [
    SellerService,
    {
      provide: 'ISellerRepository',  // 토큰
      useClass: SellerRepository,     // 구현체
    },
    BusinessValidator,
  ],
  controllers: [SellerController],
  exports: [SellerService],  // 다른 모듈에서 사용
})
export class SellerModule {}
```

**특징:**
- `@Module` 데코레이터로 모듈 정의
- `providers`에서 의존성 등록
- `imports`, `exports`로 모듈 간 의존성 관리

---

## 4. 주요 차이점 비교

### 4.1 구조 방식

| 항목 | Spring Boot | NestJS |
|------|------------|--------|
| **구조 방식** | 패키지 기반 계층 구조 | 모듈 기반 구조 |
| **계층 분리** | 패키지로 분리 (`domain`, `application`, `infrastructure`, `presentation`) | 폴더 구조 + `@Module` 데코레이터 |
| **모듈화** | 멀티 모듈 프로젝트 (Gradle/Maven) | `@Module` 데코레이터로 모듈 정의 |

### 4.2 도메인 엔티티

| 항목 | Spring Boot | NestJS |
|------|------------|--------|
| **엔티티 구조** | JPA Entity = Domain Entity (통합) | Domain Entity와 TypeORM Entity 분리 |
| **프레임워크 의존성** | JPA 어노테이션 사용 (`@Entity`, `@Column`) | 도메인 엔티티는 순수 TypeScript 클래스 |
| **매핑** | 자동 매핑 (JPA) | 수동 매핑 (`toEntity()`, `toDomain()`) |

### 4.3 Repository 패턴

| 항목 | Spring Boot | NestJS |
|------|------------|--------|
| **인터페이스 위치** | 도메인 계층 | 도메인 계층 |
| **구현체 위치** | 인프라 계층 (Adapter) | 인프라 계층 |
| **Spring Data JPA / TypeORM** | Spring Data JPA 인터페이스 사용 | TypeORM Repository 주입 (`@InjectRepository`) |
| **매핑 필요성** | 불필요 (엔티티 통합) | 필요 (도메인 ↔ TypeORM 엔티티) |

### 4.4 의존성 주입

| 항목 | Spring Boot | NestJS |
|------|------------|--------|
| **방식** | 생성자 주입 (권장) | 생성자 주입 + `@Inject()` 데코레이터 |
| **어노테이션** | `@Service`, `@Repository`, `@Component` | `@Injectable()` |
| **인터페이스 주입** | 인터페이스 직접 주입 가능 | 토큰 기반 주입 (`@Inject('토큰')`) |
| **보일러플레이트** | `@RequiredArgsConstructor` (Lombok) | 수동 작성 또는 데코레이터 |

### 4.5 트랜잭션 관리

| 항목 | Spring Boot | NestJS |
|------|------------|--------|
| **방식** | 선언적 트랜잭션 (`@Transactional`) | 수동 트랜잭션 (QueryRunner) 또는 라이브러리 |
| **AOP 지원** | 내장 (Spring AOP) | 없음 (수동 관리) |
| **읽기 최적화** | `@Transactional(readOnly = true)` | 수동 설정 |
| **복잡도** | 간단 (어노테이션만) | 복잡 (수동 관리) |

### 4.6 모듈화

| 항목 | Spring Boot | NestJS |
|------|------------|--------|
| **방식** | 멀티 모듈 프로젝트 (Gradle/Maven) | `@Module` 데코레이터 |
| **의존성 관리** | `build.gradle` / `pom.xml` | `@Module`의 `imports`, `exports` |
| **동적 구성** | 제한적 | 동적 모듈 지원 |

---

## 5. 실제 코드 비교

### 5.1 Repository 구현 비교

#### Spring Boot
```java
// 도메인: 인터페이스
public interface SellerRepository {
    Seller save(Seller seller);
    Optional<Seller> findById(UUID id);
}

// 인프라: Adapter
@Repository
@RequiredArgsConstructor
public class SellerRepositoryAdapter implements SellerRepository {
    private final SellerJpaRepository sellerJpaRepository;
    
    @Override
    public Seller save(Seller seller) {
        return sellerJpaRepository.save(seller);  // 직접 저장
    }
}
```

#### NestJS
```typescript
// 도메인: 인터페이스
export interface ISellerRepository {
  save(seller: Seller): Promise<Seller>;
  findById(id: string): Promise<Seller | null>;
}

// 인프라: 구현체
@Injectable()
export class SellerRepository implements ISellerRepository {
  constructor(
    @InjectRepository(SellerEntity)
    private readonly sellerRepo: Repository<SellerEntity>,
  ) {}
  
  async save(seller: Seller): Promise<Seller> {
    const entity = this.toEntity(seller);      // 매핑 필요
    const saved = await this.sellerRepo.save(entity);
    return this.toDomain(saved);                // 역매핑 필요
  }
}
```

### 5.2 서비스 구현 비교

#### Spring Boot
```java
@Service
@RequiredArgsConstructor
@Transactional
public class SellerService {
    private final SellerRepository sellerRepository;  // 인터페이스
    
    @Transactional(readOnly = true)
    public SellerInfoDto getASellerByUserId(UUID userId) {
        Seller seller = sellerRepository.findById(userId)
            .orElseThrow(() -> new CustomException(SELLER_NOT_FOUND));
        return SellerInfoDto.from(seller);
    }
}
```

#### NestJS
```typescript
@Injectable()
export class SellerService {
  constructor(
    @Inject('ISellerRepository')
    private readonly sellerRepository: ISellerRepository,  // 토큰 기반
  ) {}
  
  async getASellerByUserId(userId: string): Promise<SellerInfoDto> {
    const seller = await this.sellerRepository.findById(userId);
    if (!seller) {
      throw new NotFoundException('Seller not found');
    }
    return SellerInfoDto.from(seller);
  }
}
```

---

## 6. 장단점 비교

### 6.1 Spring Boot DDD

**장점:**
- ✅ **성숙한 생태계**: 오랜 기간 검증된 프레임워크
- ✅ **선언적 트랜잭션**: `@Transactional`로 간단한 관리
- ✅ **통합 엔티티**: 도메인 엔티티 = JPA Entity (매핑 불필요)
- ✅ **강력한 타입 안정성**: Java 컴파일 타임 검증
- ✅ **멀티스레딩**: JVM 기반 병렬 처리

**단점:**
- ❌ **프레임워크 의존성**: JPA 어노테이션이 도메인 계층에 침투
- ❌ **빌드 시간**: JVM 기반으로 시작 시간이 느림
- ❌ **보일러플레이트**: Java 코드가 장황함 (Lombok으로 완화)

### 6.2 NestJS DDD

**장점:**
- ✅ **도메인 순수성**: 도메인 엔티티가 프레임워크 독립적
- ✅ **빠른 개발**: TypeScript + Hot Reload
- ✅ **모듈 시스템**: `@Module`로 명확한 모듈화
- ✅ **비동기 처리**: Node.js 이벤트 루프 기반
- ✅ **유연성**: 동적 모듈, 조건부 주입 등

**단점:**
- ❌ **수동 트랜잭션**: QueryRunner로 수동 관리 필요
- ❌ **매핑 복잡도**: 도메인 ↔ TypeORM 엔티티 매핑 필요
- ❌ **런타임 한계**: Node.js 싱글 스레드 제약
- ❌ **프레임워크 침투**: `@Injectable` 등이 도메인에 침투 가능

---

## 7. 선택 가이드

### 7.1 Spring Boot DDD가 적합한 경우

- ✅ **대규모 엔터프라이즈 애플리케이션**
- ✅ **복잡한 트랜잭션 관리 필요**
- ✅ **Java/Kotlin 생태계 활용**
- ✅ **높은 안정성과 성능 요구**
- ✅ **멀티스레딩이 중요한 경우**

### 7.2 NestJS DDD가 적합한 경우

- ✅ **빠른 프로토타이핑과 개발 속도**
- ✅ **마이크로서비스 아키텍처**
- ✅ **I/O 중심 작업** (API, 메시징)
- ✅ **TypeScript/Node.js 생태계 활용**
- ✅ **도메인 순수성을 최우선으로 하는 경우**

---

## 8. Best Practice

### 8.1 Spring Boot DDD

**DO:**
```java
// ✅ 도메인 계층은 인터페이스만 정의
public interface SellerRepository {
    Seller save(Seller seller);
}

// ✅ Adapter 패턴 사용
@Repository
public class SellerRepositoryAdapter implements SellerRepository {
    // ...
}

// ✅ 생성자 주입 사용
@Service
@RequiredArgsConstructor
public class SellerService {
    private final SellerRepository sellerRepository;
}
```

**DON'T:**
```java
// ❌ 도메인 계층에 Spring 어노테이션 사용
@Service  // 도메인 계층에 사용 금지
public class Seller {
    // ...
}

// ❌ Repository 구현체를 도메인 계층에 두기
public class SellerRepositoryImpl implements SellerRepository {
    // ...
}
```

### 8.2 NestJS DDD

**DO:**
```typescript
// ✅ 도메인 엔티티는 순수 클래스
export class Seller {
  // 프레임워크 독립적
}

// ✅ TypeORM Entity는 인프라 계층에만
@Entity('seller')
export class SellerEntity {
  // ...
}

// ✅ Repository에서 매핑
private toDomain(entity: SellerEntity): Seller {
  // ...
}
```

**DON'T:**
```typescript
// ❌ 도메인 엔티티에 TypeORM 어노테이션 사용
@Entity('seller')  // 도메인 계층에 사용 금지
export class Seller {
  // ...
}

// ❌ 도메인 계층에 @Injectable 사용 (가능하지만 주의)
@Injectable()  // 가능하지만 프레임워크 침투
export class Seller {
  // ...
}
```

---

## 마무리

**핵심 포인트:**

1. **구조 방식**: Spring Boot는 패키지 기반, NestJS는 모듈 기반
2. **도메인 엔티티**: Spring Boot는 JPA Entity와 통합, NestJS는 분리
3. **Repository**: Spring Boot는 Adapter 패턴, NestJS는 매핑 필요
4. **트랜잭션**: Spring Boot는 선언적, NestJS는 수동 관리
5. **의존성 주입**: Spring Boot는 인터페이스 직접 주입, NestJS는 토큰 기반

두 프레임워크 모두 DDD 원칙을 적용할 수 있지만, **Spring Boot는 통합과 편의성**에, **NestJS는 순수성과 유연성**에 강점이 있습니다. 프로젝트의 요구사항과 팀의 경험에 따라 적절한 프레임워크를 선택하는 것이 중요합니다. 🚀

다음 글에서는 **Kubernetes IaC 도구들(Kustomize, Helm, Terraform, Pulumi)의 특징과 사용 상황**을 비교해보겠습니다.
