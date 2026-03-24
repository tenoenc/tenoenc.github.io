---
layout: post
title: "[System Design] Layered에서 Vertical Slice까지"
date: 2026-01-04 22:18 +0900
math: true
categories:
- Backend Engineering
- System Design
tags:
- System Design
- Layered Architecture
- Hexagonal Architecture
- Vertical Slice Architecture
image:
  path: /assets/img/2026-01-05-02-10-54.png
---

아키텍처를 논할 때 보통 모놀리식이나 마이크로서비스 같은 배포 구조를 떠올리지만, 진정한 의미의 아키텍처는 단일 프로세스 내의 코드 조직화(Internal Structure)에서 시작됩니다.

내부 아키텍처의 목표는 패턴의 적용 자체가 아니라 개발자가 코드를 읽을 때의 인지 부하(Cognitive Load)를 줄이고, 관련 로직을 가깝게 배치하여 응집도(Cohesion)를 높이는 것입니다.

## 🧩 Layered Architecture

자바 생태계에서 Layered Architecture(N-Tier)는 보편적인 구조입니다. `controller`, `service`, `repository` 패키지 구성이 일반적이며, 마크 리차즈(Mark Richards)는 이를 "사실상의 표준"이라 정의했습니다. 익숙한 구조로 팀의 조직 구조와 일치하여 초기 개발 속도가 빠르지만 단점이 존재합니다.

### 관심 범위의 축소

계층을 나누는 본질적인 가치는 마틴 파울러(Martin Fowler)가 정의한 '관심 범위의 축소'에 있습니다. 

- UI 코드를 작성할 때는 복잡한 SQL 조인 쿼리를 고려하지 않아도 된다.
- 비즈니스 로직을 구현할 때는 데이터가 어떤 형식으로 렌더링될지 신경 쓰지 않는다.

개발자가 한 번에 다뤄야 할 정보량을 제한하고 하나의 문제에만 집중하게 만드는 것이 이 구조의 핵심입니다.

### 데이터베이스 주도 개발

Layered Architecture는 의존성 흐름이 상위(Presentation)에서 하위(Persistence)로 향하므로 최하단의 데이터베이스가 시스템의 중심이 됩니다.

일반적인 개발 프로세스는 다음과 같은 순서로 진행됩니다.
1. ERD 설계: 테이블과 관계를 우선 정의한다.
2. Entity 생성: 테이블과 1:1로 매핑되는 클래스를 만든다.
3. CRUD 구현: 해당 Entity를 처리하는 Repository, Service, Controller를 순차적으로 생성한다.

이 과정에서 객체 간의 협력보다는 데이터 중심의 빈혈 도메인 모델(Anemic Domain Model)이 생성되기 쉽습니다. 결국 서비스 계층은 도메인 로직을 수행하는 대신 데이터를 가공해 전달하는 트랜잭션 스크립트(Transaction Script) 위주로 구성되며, DB 스키마 변경 시 상위 계층까지 수정해야 하는 강한 결합이 발생합니다.

### Anti-Pattern: 아키텍처 싱크홀

요청이 레이어를 통과할 때 비즈니스 가치를 더하지 못하고 하위 레이어로 단순 전달(Pass-through)만 하는 상황을 아키텍처 싱크홀(Architecture Sinkhole)이라 합니다.

```java
@Service
@RequiredArgsConstructor
public class ProductService {
    private final ProductRepository productRepository;

    @Transactional(readOnly = true)
    public ProductDto getProduct(Long id) {
        return productRepository.findById(id)
                .map(Product::toDto)
                .orElseThrow(() -> new EntityNotFoundException());
    }
}
```

위 사례에서 서비스는 리포지토리의 프록시 역할만 수행합니다.

> **아키텍처 선택과 싱크홀 비율**
> 
> 마크 리차즈는 "전체 요청의 20% 정도는 허용되지만, 80% 이상이 싱크홀이라면 이 아키텍처는 잘못된 선택"이라고 분석합니다.
{: .prompt-warning }

레이어 구조를 유지하기 위해 발생하는 객체 변환 및 관리 비용은 설계의 효율성 관점에서 검토가 필요합니다.

## 🧩 Hexagonal Architecture

![](/assets/img/2026-01-05-02-39-45.png)

Layered 아키텍처는 상위 계층이 하위 계층에 의존하는 구조적 특성상 비즈니스 로직이 데이터베이스 기술(JPA, MyBatis 등)에 종속되기 쉽습니다. 앨리스테어 콕번(Alistair Cockburn)은 이러한 의존성 방향 문제를 해결하기 위해 내부와 외부를 분리하는 Hexagonal Architecture(Ports and Adapters)를 제안했습니다.

### 의존성 역전

이 아키텍처의 핵심 철학은 내부와 외부의 철저한 격리입니다.
- **Inside (Domain Hexagon)**: 순수한 비즈니스 로직이 위치합니다. 외부의 기술적 세부사항을 알지 못하며, 프레임워크에 의존하지 않는 POJO(Plain Old Java Object) 상태를 유지합니다.
- **Outside (Infrastructure)**: 웹 브라우저, 데이터베이스, 외부 API, 메시지 큐 등 애플리케이션 외부의 기술적 요소들입니다.

여기서 의존성 역전(DIP)이 적용됩니다. 도메인이 포트(Port)라는 인터페이스를 정의하면, 인프라 계층이 어댑터(Adapter)를 통해 이를 구현합니다. 결과적으로 의존성의 화살표가 외부에서 내부로 향하게 됩니다.

> **Why Hexagon?**
> 
> 육각형이라는 모양은 애플리케이션이 여러 면(Port)을 통해 외부와 소통할 수 있음을 시각적으로 표현합니다. 특정 기술에 국한되지 않고 다양한 방식으로 외부와 연결될 수 있다는 의미를 담고 있습니다.
{: .prompt-info }

### Netflix의 유연함

이 구조는 외부 시스템을 교체할 때 효율적입니다. 넷플릭스(Netflix) 엔지니어링 팀은 스튜디오 애플리케이션에 이 아키텍처를 도입하여 데이터 소스를 기존 JSON API에서 GraphQL로 변경하는 작업을 수행했습니다. 헥사고날 구조 덕분에 도메인 로직을 수정하지 않고 새로운 어댑터(Driven Adapter)를 구현하여 교체하는 방식으로 마이그레이션을 완료할 수 있었습니다.

### 매핑 지옥

도메인의 순수성을 유지하는 과정에서 인지 부하와 매핑 비용이 발생합니다. 데이터를 저장하는 간단한 기능에서도 다음과 같은 단계를 거치게 됩니다.
1. **Web DTO**: 컨트롤러가 요청을 받는 객체
2. **Command 객체**: 입력 포트로 전달하기 위한 변환 객체
3. **Domain Entity**: 비즈니스 로직을 수행하는 핵심 객체
4. **JPA Entity**: 영속성 계층에서 사용되는 변환 객체

필드 하나를 추가할 때마다 여러 클래스를 수정하고 매핑 코드를 작성해야 하므로, 이를 '매핑 지옥'이라 부르기도 합니다. 복잡도가 높은 핵심 도메인에서는 이러한 비용이 정당화될 수 있으나, 단순한 CRUD 시스템에서는 오버 엔지니어링이 될 가능성이 높습니다.

## 🧩 Vertical Slice Architecture

Layered와 Hexagonal 아키텍처는 코드를 기술적 계층(Horizontal Layers)에 따라 분류한다는 공통점이 있습니다. 지미 보가드(Jimmy Bogard)는 기능 변경이 UI, 로직, DB 전반에 걸쳐 수직(Vertical)으로 발생함에도 코드를 수평으로 관리하는 방식에 의문을 제시했습니다. 이러한 문제의식에서 출발한 설계 방식이 Vertical Slice Architecture입니다.

### 변경의 축을 따르라

이 아키텍처의 핵심은 응집도의 재정의입니다. 기존에는 같은 종류의 코드(Controller, Service 등)를 모으는 것을 응집이라 보았으나, Vertical Slice는 함께 변경되는 코드를 모으는 것이 진정한 응집도라고 주장합니다.

- **Before (Layered):** 특정 기능을 수정하기 위해 여러 기술 계층 패키지를 탐색해야 하므로 응집도가 낮아집니다.
- **After (Slice):** `features/registerUser` 패키지 안에 해당 기능에 필요한 Controller, Command, Validator, Repository 등이 모두 포함됩니다.

관련된 모든 문맥이 하나의 폴더(Slice) 안에 격리되므로 탐색 비용이 줄어듭니다. 이는 기술적 계층이 아닌 비즈니스 역량에 따른 격리를 의미합니다.

### 패턴으로의 리팩토링

Vertical Slice의 특징은 구현의 유연함입니다. 모든 슬라이스가 동일한 구조를 가질 필요가 없으며, 문제의 복잡도에 따라 최적의 방식을 선택합니다.

- **단순 조회 (Simple Query):** 별도의 DTO 변환이나 복잡한 도메인 모델 없이 Controller가 직접 DB를 호출하여 효율을 높일 수 있습니다.
- **복잡한 비즈니스 (Complex Command):** 풍부한 도메인 모델(Rich Domain Model)을 사용하여 엄격한 캡슐화와 테스트를 적용합니다.

각 기능의 특성에 맞춰 불필요한 추상화를 제거하고 실용성을 확보하는 접근법입니다.

### 중복을 허용하라

공통 로직이나 중복 코드에 대해서는 '우발적 중복(Accidental Duplication)'을 허용하는 방식을 취합니다. 섣부른 공통화(DRY 원칙 준수)는 서로 다른 변경 주기를 가진 기능을 하나의 코드로 묶어 '우발적 결합(Accidental Coupling)'을 초래하기 때문입니다.

공통 코드를 수정했을 때 예상치 못한 기능에서 오류가 발생하는 상황을 방지하기 위해, 완벽한 격리가 유리하다고 판단되면 코드 중복을 감수합니다. 코드를 공유하는 것보다 각 기능이 독립적으로 유지보수될 수 있는 환경을 만드는 데 집중합니다.

## Architecture as Code

아무리 완벽한 아키텍처를 설계해도 시간이 지나면 코드는 점차 엉키기 마련입니다. 아키텍처 제약 조건은 개발자의 기억이나 문서가 아니라, 실패하는 단위 테스트(Failing Unit Test)를 통해 강제되어야 합니다.

자바 진영의 ArchUnit은 컴파일된 바이트코드(Bytecode)를 분석하여 의존성 규칙을 검증하는 도구입니다.

### Layered Architecture

Layered 아키텍처에서 가장 흔하게 발생하는 문제는 계층 건너뛰기(Bypass)입니다. Controller가 Service를 거치지 않고 Repository를 직접 참조하는 순간, 트랜잭션 관리와 비즈니스 로직의 캡슐화가 깨집니다.

이러한 위반 사항을 코드 리뷰만으로 찾아내기는 어렵습니다. 하지만 이를 코드로 정의해두면 빌드 타임에 즉시 감지할 수 있습니다.

```java
@AnalyzeClasses(packages = "com.teno.lab")
public class LayeredArchitectureTest {

    @ArchTest
    static final ArchRule layer_boundaries_are_respected = layeredArchitecture()
            .consideringOnlyDependenciesInLayers()
            // 1. 레이어 정의: 패키지 구조와 매핑
            .layer("Controller").definedBy("..controller..")
            .layer("Service").definedBy("..service..")
            .layer("Repository").definedBy("..repository..")

            // 2. 제약 조건: 화살표는 위에서 아래로만 흐른다
            .whereLayer("Controller").mayNotBeAccessedByAnyLayer()
            .whereLayer("Service").mayOnlyBeAccessedByLayers("Controller")
            .whereLayer("Repository").mayOnlyBeAccessedByLayers("Service");
}
```

이 테스트를 통해 Controller에서 Repository를 실수로 임포트하는 상황을 CI 파이프라인 단계에서 차단할 수 있습니다.

### Vertical Slice Architecture

Vertical Slice 아키텍처의 주요 리스크는 기능 간의 의존성이 엉키는 현상입니다. 특정 기능(A)이 다른 기능(B)의 내부 클래스를 직접 참조하기 시작하면 모듈로서의 격리가 파괴됩니다.

ArchUnit의 `SlicesRuleDefinition`을 사용하면 슬라이스 간의 독립성을 검증할 수 있습니다.

```java
@AnalyzeClasses(packages = "com.teno.lab")
public class SliceArchitectureTest {

    @ArchTest
    static final ArchRule slices_should_be_free_of_cycles =
        SlicesRuleDefinition.slices()
            .matching("com.teno.lab.features.(*)..") // features 하위 패키지를 하나의 슬라이스로 간주
            .should().beFreeOfCycles(); // 순환 참조 금지
            
    @ArchTest
    static final ArchRule features_should_not_depend_on_other_features =
        classes().that().resideInAPackage("..features..")
            .should().onlyAccessClassesThat()
            .resideInAnyPackage("java..", "javax..", "..shared..", "..features.."); 
            // 더 엄격하게 Shared와 JDK 외에는 서로 참조 금지 설정 가능
}
```

### Living Documentation

작성된 아키텍처 테스트는 그 자체로 살아있는 문서(Living Documentation) 역할을 수행합니다. 엔지니어링은 주관적인 믿음이 아니라 시스템을 통해 문제를 해결하는 과정입니다. 아키텍처 역시 시스템을 통한 검증이 뒷받침되어야 유지될 수 있습니다.

## Summary

모든 상황에 적합한 단일 아키텍처는 존재하지 않습니다. 프로젝트의 성격과 복잡도, 그리고 팀의 상황에 맞춰 최적의 구조를 선택해야 합니다.

### Selection Criteria

| Architecture       | Best Fit For                                               | Key Constraint                                                   |
| :----------------- | :--------------------------------------------------------- | :--------------------------------------------------------------- |
| **Layered**        | 단순 CRUD 위주의 시스템, 빠른 초기 개발이 필요한 경우      | 비즈니스 복잡도가 증가할수록 계층 간 결합도와 유지보수 비용 상승 |
| **Hexagonal**      | 핵심 도메인 보호가 중요하고 외부 기술 교체가 잦은 프로젝트 | 높은 구현 비용 및 객체 매핑 오버헤드 발생                        |
| **Vertical Slice** | 기능 단위의 빈번한 변경과 유연한 구현 방식이 필요한 경우   | 개발자의 리팩토링 역량과 코드 격리에 대한 판단력 요구            |

### Hybrid Modular Monolith

실무에서는 전체 시스템을 모듈형 모놀리스(Modular Monolith)로 구성하되, 모듈의 중요도와 성격에 따라 내부 아키텍처를 다르게 적용하는 전략이 유효합니다.

> **모듈별 아키텍처 적용 전략**
> 
> 도메인 로직이 복잡한 핵심 모듈은 헥사고날 아키텍처로 기술적 상세를 격리하고, 단순 조회나 지원 성격의 모듈은 수직 슬라이스로 생산성을 확보하는 방식입니다. 시스템 전반의 일관성보다는 실용적인 설계가 우선되어야 합니다.
{: .prompt-tip }

## References

* [[Martin Fowler] Presentation Domain Data Layering](https://martinfowler.com/bliki/PresentationDomainDataLayering.html)
* [[Alistair Cockburn] Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture)
* [[Netflix TechBlog] Ready for changes with Hexagonal Architecture](https://netflixtechblog.com/ready-for-changes-with-hexagonal-architecture-b315ec967749)
* [[Jimmy Bogard] Vertical Slice Architecture](https://www.jimmybogard.com/vertical-slice-architecture/)
* [[CodeOpinion] Vertical Slice Architecture isn't technical](https://codeopinion.com/vertical-slice-architecture-isnt-technical/)