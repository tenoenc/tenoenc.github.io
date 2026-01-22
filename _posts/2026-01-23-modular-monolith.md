---
layout: post
title: "[System Design] Modular Monolith"
date: 2026-01-23 06:42 +0900
math: true
categories:
- Backend Engineering
- System Design
tags:
- System Design
- Modular Monolithic Architecture
image:
    path: /assets/img/2026-01-23-07-14-00.png
---

## 🧩 Module

대규모 시스템을 구축할 때 흔히 저지르는 실수는 모듈을 단순히 '기능별로 분류된 폴더'로 취급하는 것입니다. 그러나 엔지니어링 관점에서 모듈은 **추상화의 벽(Abstraction Wall)**입니다. 내부의 복잡성을 외부로 유출하지 않으면서도, 시스템의 전체적인 정합성을 유지하기 위한 논리적 요새여야 합니다. 모듈화가 실패한 모놀리스는 결국 모든 객체가 서로를 참조하는 '커다란 진흙 덩어리(Big Ball of Mud)'로 전락하며, 이는 곧 수정 하나가 시스템 전체의 회귀 테스트를 강제하는 재앙으로 이어집니다.

### Encapsulation

진정한 의미의 모듈화는 캡슐화에서 시작됩니다. 캡슐화는 단순히 필드를 `private`으로 선언하는 행위를 넘어, 모듈 내부의 **불변성(Invariants)**을 보호하고 구현 세부 사항을 숨기는 전략적 선택입니다. 모듈 내부의 도메인 로직이나 데이터 구조가 외부로 노출되는 순간, 그 모듈은 더 이상 독립적으로 변경될 수 없습니다. 외부의 수많은 참조자가 해당 구조에 의존하게 되기 때문입니다.

우리는 모듈 내부를 'Black Box'로 유지해야 합니다. 외부에서는 모듈이 어떤 데이터베이스 라이브러리를 쓰는지, 어떤 복잡한 알고리즘으로 결과값을 도출하는지 알 필요가 없습니다. 오직 약속된 인터페이스를 통해서만 상호작용해야 합니다. 이러한 엄격한 캡슐화는 개발자가 모듈 내부 코드를 리팩토링할 때 '외부에 미칠 영향'을 고민하지 않게 함으로써 개발 속도를 비약적으로 향상시킵니다.

![](/assets/img/2026-01-23-06-53-54.png)

### Public API

모듈의 Public API는 해당 모듈이 세상과 소통하는 유일한 창구입니다. 이 창구는 최대한 좁고 견고하게 설계되어야 합니다. Java 진영에서는 패키지 가시성(Package Visibility)을 활용하거나 Java 9+의 Module System을 통해 명시적으로 `exports`할 대상을 지정할 수 있습니다. Spring Modulith 같은 프레임워크를 사용하면 특정 패키지(예: `.internal`) 하위의 클래스들을 외부 모듈에서 참조할 경우 테스트 단계에서 실패를 발생시켜 물리적인 격리를 강제할 수도 있습니다.

좋은 API 설계는 사용하기 쉽고, 오용하기 어려워야 합니다. 이를 위해 모듈 간 통신에는 내부 엔티티(Entity)를 그대로 노출하지 않고, 오직 통신을 위한 순수한 데이터 구조인 DTO(Data Transfer Object)를 사용하는 것이 원칙입니다. 엔티티의 변경이 API 스펙의 변경으로 전이되는 것을 차단하는 방화벽 역할을 하기 때문입니다.

```java
// 1. 주문 모듈의 공개 API (외부 모듈에서 참조 가능)
package com.example.order;

import lombok.Value;

public interface OrderService {
    void placeOrder(OrderRequest request);
}

@Value
public class OrderRequest {
    String productId;
    int quantity;
}

// 2. 주문 모듈의 내부 구현 (외부 모듈에서 참조 불가 - 캡슐화)
package com.example.order.internal;

import com.example.order.OrderRequest;
import org.springframework.stereotype.Service;

@Service
class OrderProcessor { // package-private으로 선언하여 외부 접근 차단
    public void validate(OrderRequest request) {
        // 복잡한 내부 검증 로직
    }
}
```

### Domain Autonomy

각 모듈은 자신의 도메인 영역에 대해 절대적인 자율성을 가집니다. 이는 모듈이 스스로의 상태를 관리하고, 자신의 비즈니스 규칙을 완결성 있게 처리함을 의미합니다. 예를 들어 '주문(Order)' 모듈은 '재고(Inventory)' 모듈의 데이터베이스 테이블을 직접 조회해서는 안 됩니다. 재고 확인이 필요하다면 재고 모듈이 제공하는 API를 호출하거나, 도메인 이벤트를 통해 상태 변화를 통지받아야 합니다.

이러한 자율성은 팀 단위의 병렬 개발을 가능하게 하는 핵심 동력입니다. 각 팀은 자신이 맡은 모듈의 경계 안에서 자유롭게 기술 스택을 최적화하거나 내부 구조를 변경할 수 있습니다. 모듈이 자율적일수록, 시스템은 마치 레고 블록처럼 유연하게 조립되고 확장될 수 있는 유기체로 거듭납니다.

> **모듈화의 비용과 가치**
> 
> 모든 코드 뭉치를 모듈로 쪼개는 것은 과잉 엔지니어링일 수 있습니다. 모듈화는 인터페이스 설계와 데이터 변환이라는 추가적인 비용을 수반합니다. 
> 
> 하지만 시스템의 복잡도가 지수적으로 증가하는 시점에서, 이 '구조적 비용'은 '유지보수 재앙'을 막기 위한 가장 저렴한 보험료가 됩니다.
{: .prompt-info }

## 🧩 Monolith

모듈형 모노리스 아키텍처에서 **Monolith**는 논리적으로 분리된 모듈들이 '물리적으로는 하나로 통합되어 배포됨'을 의미합니다. 이는 마이크로서비스 아키텍처(MSA)가 강제하는 네트워크 경계(Network Boundary) 대신, 프로세스 내 통신(In-process Communication)을 선택함으로써 운영 복잡도를 제어하는 전략적 결정입니다. 모듈화된 코드가 하나의 실행 단위로 묶일 때, 시스템은 분산 시스템의 고질적인 문제인 네트워크 지연(Latency)과 부분 실패(Partial Failure)의 공포에서 해방됩니다.

### Deployment Unit

모노리스의 가장 강력한 무기는 **단일 배포 유닛(Single Deployment Unit)**이 주는 단순함입니다. 모든 도메인 로직이 하나의 바이너리 혹은 아티팩트로 빌드되어 배포되므로, 지속적 통합 및 배포(CI/CD) 파이프라인이 극도로 간결해집니다. 서비스 간 버전 불일치로 인한 '의존성 지옥'이나 복잡한 서비스 메시(Service Mesh) 설정 없이도 시스템을 안정적으로 운영할 수 있습니다.

또한, 모듈 간 호출이 인메모리 함수 호출(In-memory call)로 이루어지기 때문에 직렬화/역직렬화에 따른 CPU 오버헤드가 발생하지 않습니다. 이는 고성능이 요구되는 데이터 집약적 작업에서 압도적인 효율성을 제공합니다. 개발자는 분산 트랜잭션의 복잡성을 고민하는 대신, 단일 데이터베이스 트랜잭션 내에서 모듈 간 데이터 정합성을 보장하는 데 집중할 수 있습니다.

![](/assets/img/2026-01-23-06-54-04.png)

### Resource Contention

단일 프로세스 구조는 효율적이지만, **자원 경합(Resource Contention)**이라는 치명적인 트레이드오프를 수반합니다. 모든 모듈이 동일한 힙 메모리(Heap Memory)와 CPU 코어를 공유하기 때문에, 특정 모듈의 메모리 누수나 무거운 연산이 시스템 전체의 가용성을 위협할 수 있습니다. 이를 '폭발 반경(Blast Radius)' 문제라고 부릅니다.

마이크로서비스였다면 특정 서비스만 크래시가 발생하고 끝날 일이, 모노리스에서는 단 하나의 잘못된 모듈로 인해 전체 시스템이 다운되는 연쇄 반응을 일으킵니다. 따라서 모듈형 모노리스를 지탱하기 위해서는 모듈별 리소스 사용량에 대한 정밀한 모니터링과, 특정 모듈이 전체 스레드 풀을 점유하지 못하도록 차단하는 서킷 브레이커(Circuit Breaker) 혹은 벌크헤드(Bulkhead) 패턴의 적용이 필수적입니다.

> **단일성인가, 격리인가?**
> 
> 모노리스를 선택한다는 것은 '운영의 편의성'을 위해 '물리적 격리'를 포기하는 선택입니다. 
> 
> 만약 특정 모듈의 트래픽이 다른 모듈보다 수만 배 이상 높거나, 독자적인 스케일링 정책이 절실하다면 그 지점이 바로 모노리스의 경계가 무너지고 MSA로의 전환을 고려해야 할 시점입니다.
{: .prompt-warning }

## 🧩 Boundary

모듈형 모노리스의 성패는 코드를 어떻게 나누느냐가 아니라, 나뉜 코드 사이에 얼마나 견고한 **경계(Boundary)**를 세우느냐에 달려 있습니다. 경계가 모호한 모듈화는 결국 코드만 복잡해진 '스파게티 모노리스'로 회귀할 뿐입니다. 엔지니어링 관점에서 경계는 도메인의 의미론적 한계선인 동시에, 물리적인 코드 참조를 차단하는 기술적 장벽입니다.

### Bounded Context

모듈의 경계를 설정하는 가장 강력한 기준은 도메인 주도 설계(DDD)의 **Bounded Context**입니다. 동일한 용어라도 맥락에 따라 그 의미가 완전히 달라질 수 있음을 인정하는 것이 핵심입니다. 예를 들어, '사용자(User)'라는 객체는 결제 모듈에서는 '결제 수단을 가진 구매자'이지만, 배송 모듈에서는 '수취 주소를 가진 수령인'입니다.

하나의 거대한 전역 모델을 만들려고 시도하는 순간 경계는 붕괴됩니다. 대신, 각 모듈 내부에 해당 컨텍스트에만 최적화된 모델을 구축하고, 모듈 간 데이터를 주고받을 때만 최소한의 정보로 매핑하는 전략이 필요합니다. 각 모듈이 자신만의 언어와 규칙을 가진 독립된 영토임을 인정할 때, 비로소 진정한 의미의 도메인 자율성이 확보됩니다.

![](/assets/img/2026-01-23-06-54-36.png)

### Package Visibility

설계상으로 경계를 확정했다면, 이를 코드로 강제해야 합니다. 대부분의 프로그래밍 언어는 **패키지 가시성(Package Visibility)** 기능을 제공하여 모듈 외부에서 접근 가능한 클래스를 제한할 수 있습니다. 예를 들어, Java의 `public` 키워드를 남발하지 않고 `package-private`을 활용하여 모듈 내부의 구현체들을 숨기는 것이 그 시작입니다.

최근의 모던 프레임워크들은 여기서 한 발 더 나아가 아키텍처 규칙을 테스트 코드로 검증합니다. 특정 패키지가 허용되지 않은 다른 패키지의 클래스를 직접 호출하거나 상속받는 행위를 빌드 단계에서 차단하는 것입니다. 이는 시니어 개발자가 코드 리뷰로 일일이 확인하던 '경계 침범'을 시스템이 자동으로 감지하게 함으로써, 시간이 흐름에 따라 아키텍처가 부식(Erosion)되는 것을 방지하는 강력한 가드레일이 됩니다.

> **누수되는 추상화의 위험**
> 
> 경계가 무너지는 가장 흔한 징후는 모듈 내부의 예외(Exception) 클래스나 데이터베이스 스키마와 밀접하게 연관된 객체가 외부로 노출되는 것입니다. 
> 
> 외부 모듈이 내부의 예외를 `catch`해야 하거나 내부 엔티티의 필드 구조를 알아야 한다면, 이미 그 경계는 제 기능을 상실한 상태입니다.
{: .prompt-warning }

## 🧩 Interface

모듈 간의 결합을 느슨하게 유지하면서도 협업을 가능하게 하는 유일한 수단은 **인터페이스(Interface)**입니다. 모듈형 모노리스에서 인터페이스는 단순히 '메서드 시그니처의 집합'이 아니라, 두 도메인이 서로에게 동의한 **명시적 계약(Explicit Contract)**입니다. 이 계약이 정교할수록 시스템은 거대한 진흙 덩어리가 아닌, 조립 가능한 부품들의 집합체로 남을 수 있습니다.

### Dependency Inversion

모듈 간의 의존성 방향은 아키텍처의 건전성을 결정합니다. 흔히 저지르는 실수는 상위 수준의 정책 모듈(예: 주문)이 하위 수준의 상세 모듈(예: 알림)에 직접 의존하는 것입니다. 이는 알림 모듈의 작은 변경이 주문 모듈의 재빌드와 재배포를 강제하는 결과를 낳습니다.

이를 해결하는 것이 **의존성 역전 원칙(DIP)**입니다. 주문 모듈은 자신이 필요한 기능을 인터페이스로 정의하고, 알림 모듈은 이 인터페이스를 구현합니다. 이로써 의존성의 방향은 제어 흐름과 반대로 역전되며, 주문 모듈은 알림 모듈의 구체적인 구현(SMS인지, 이메일인지)으로부터 완벽하게 자유로워집니다. 모듈형 모노리스에서 DIP는 물리적인 프로젝트 분리 없이도 런타임 결합도를 낮추는 가장 효과적인 도구입니다.

```java
// [Order Module] - 도메인 계층에 위치한 인터페이스
package com.example.order.domain;

public interface NotificationPort {
    void send(String message, String recipient);
}

// [Order Module] - 서비스 로직은 인터페이스에만 의존
package com.example.order.application;

import com.example.order.domain.NotificationPort;
import lombok.RequiredArgsConstructor;

@RequiredArgsConstructor
public class OrderService {
    private final NotificationPort notificationPort;

    public void completeOrder(String orderId) {
        // ... 주문 처리 ...
        notificationPort.send("주문이 완료되었습니다.", "user@example.com");
    }
}

// [Notification Module] - 인터페이스의 실제 구현체
package com.example.notification.infrastructure;

import com.example.order.domain.NotificationPort;
import org.springframework.stereotype.Component;

@Component
public class EmailNotificationAdapter implements NotificationPort {
    @Override
    public void send(String message, String recipient) {
        // 실제 이메일 발송 SMTP 로직
    }
}
```

### DTO Mapping

모듈 경계를 넘나드는 데이터는 반드시 그 형태가 변환되어야 합니다. 모듈 내부에서 사용하는 풍부한 도메인 엔티티(Entity)를 외부 모듈에 직접 노출하는 것은 아키텍처의 자살 행위와 같습니다. 엔티티는 데이터베이스 스키마나 비즈니스 규칙에 강하게 결합되어 있어, 이를 외부에서 참조하는 순간 해당 엔티티를 사용하는 모든 모듈이 하나의 거대한 전역 상태를 공유하게 되기 때문입니다.

따라서 인터페이스를 통해 데이터를 주고받을 때는 오직 데이터 전달만을 목적으로 하는 **DTO(Data Transfer Object)**를 사용해야 합니다. 각 모듈은 인터페이스 호출 직전에 자신의 내부 모델을 DTO로 변환(Mapping)하고, 받는 쪽 역시 DTO를 자신의 내부 모델로 다시 변환합니다. 이 변환 과정은 번거로워 보일 수 있지만, 각 모듈의 내부 구조를 독립적으로 진화시킬 수 있게 해주는 '디커플링 비용'으로서 충분한 가치를 가집니다.

> **공유 커널의 유혹**
> 
> 여러 모듈에서 공통으로 쓰이는 `Money`나 `Address` 같은 값 객체(Value Object)를 'Common' 패키지에 몰아넣고 싶은 유혹이 생깁니다. 
> 
> 하지만 이 'Common' 패키지가 비대해질수록 모든 모듈이 이 패키지에 의존하게 되어, 결국 또 다른 형태의 강결합을 유발합니다. 정말 공통적인 유틸리티가 아니라면, 차라리 각 모듈이 필요한 정보를 중복해서 정의하는 것이 아키텍처 관점에서는 더 안전합니다.
{: .prompt-tip }

## 🧩 Dependency Management

모듈화된 시스템의 수명을 결정하는 가장 결정적인 지표는 의존성의 복잡도입니다. 아무리 도메인을 잘 나누고 인터페이스를 정교하게 설계했더라도, 모듈 간의 의존성 흐름이 무질서하게 엉키기 시작하면 시스템은 거대한 스파게티로 회귀합니다. 엔지니어링 실무에서 의존성 관리는 단순히 라이브러리를 추가하는 행위가 아니라, 시스템의 **비순환 방향성 그래프(DAG)**를 유지하기 위한 끊임없는 투쟁입니다.

### Cyclic Dependency

순환 참조(Cyclic Dependency)는 모듈화의 가장 큰 적입니다. 모듈 A가 B에 의존하고, B가 다시 A에 의존하는 구조가 형성되는 순간, 두 모듈은 논리적으로 하나가 되어버립니다. 이를 '순환 참조의 저주'라고 부릅니다. 이 상태에서는 특정 모듈만 떼어내어 테스트하거나 재사용하는 것이 불가능해지며, 코드 변경의 영향 범위가 원을 타고 무한히 확장됩니다.

순환 참조를 끊어내는 가장 표준적인 방법은 앞서 언급한 의존성 역전(DIP)을 활용하거나, 공통된 로직을 제3의 모듈로 추출하는 것입니다. 하지만 더 중요한 것은 순환 참조가 발생하기 전에 이를 감지하고 차단하는 시스템적 장치입니다. 모듈형 모노리스에서는 컴파일 타임 혹은 빌드 타임에 의존성 그래프를 분석하여 순환 구조가 발견되면 즉시 빌드를 실패시키는 엄격한 정책이 필요합니다.

![](/assets/img/2026-01-23-06-54-48.png)

### Architecture Guardrail (ArchUnit)

사람의 의지는 믿을 대상이 아닙니다. 프로젝트에 새로 합류한 개발자가 실수로 경계를 침범하거나 편리함을 위해 잘못된 의존성을 추가하는 것은 흔한 일입니다. 이를 방지하기 위해 우리는 **아키텍처 가드레일(Architecture Guardrail)**을 자동화해야 합니다.

Java 진영의 ArchUnit은 아키텍처 규칙을 유닛 테스트 코드로 작성할 수 있게 해주는 강력한 도구입니다. "주문 모듈은 배송 모듈의 내부 구현체에 의존해서는 안 된다"라거나 "모든 Controller는 Service 계층만 호출해야 한다"와 같은 규칙을 코드로 명시합니다. 이러한 'Architecture as Code' 접근 방식은 코드 리뷰에 소요되는 에너지를 비즈니스 로직에 집중시키고, 아키텍처의 부식을 원천 차단합니다.

```java
package com.example.architecture;

import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.ArchRule;

import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;

@AnalyzeClasses(packages = "com.example")
public class ModuleBoundaryTest {

    // 규칙: 주문(Order) 모듈은 결제(Payment) 모듈의 'internal' 패키지를 직접 참조할 수 없음
    @ArchTest
    static final ArchRule order_should_not_depend_on_payment_internal =
        noClasses().that().resideInAPackage("..order..")
        .should().dependOnClassesThat().resideInAPackage("..payment.internal..");

    // 규칙: 모든 모듈은 서로의 'internal' 패키지를 참조해서는 안 됨 (Public API만 허용)
    @ArchTest
    static final ArchRule modules_should_respect_internal_encapsulation =
        noClasses().that().resideInAPackage("..internal..")
        .should().beAccessed().byAnyPackage("..internal..");
}
```

> **의존성 그래프의 단순화**
> 
> 모듈의 개수가 늘어날수록 의존성 관리는 기하급수적으로 어려워집니다. 
> 
> 이를 해결하기 위해 상위 모듈이 하위 모듈의 존재를 알지 못하게 하는 '이벤트 주도 통신(Event-driven Communication)'을 적극적으로 도입하십시오. 직접적인 메서드 호출 대신 발행-구독(Pub-Sub) 모델을 사용하면 의존성 그래프의 연결선을 획기적으로 줄일 수 있습니다.
{: .prompt-tip }

## Logical Isolation in Shared Database

모듈형 모노리스 아키텍처에서 가장 해결하기 까다로운 지점은 데이터베이스입니다. 코드는 깔끔하게 나눴더라도, 모든 모듈이 하나의 거대한 공유 데이터베이스(Shared Database)에서 서로의 테이블을 마구잡이로 조인(Join)하고 있다면, 그것은 '분산된 코드'를 가진 '거대한 데이터 덩어리'일 뿐입니다. 진정한 격리는 스토리지 계층에서도 논리적인 벽을 세울 때 완성됩니다.

### Schema Isolation

데이터 격리의 첫걸음은 물리적으로는 하나의 DB 인스턴스를 사용하되, 논리적으로는 각 모듈만이 자신의 테이블에 접근할 수 있도록 권한을 제한하는 것입니다. 가장 단순한 방법은 테이블 이름에 접두어(e.g., `ord_`, `inv_`)를 붙이는 것이지만, 더 강력한 방법은 데이터베이스의 **스키마(Schema)** 기능을 활용하는 것입니다.

각 모듈은 자신만의 전용 스키마를 가지며, 타 모듈의 스키마에 있는 테이블은 직접 조회할 수 없습니다. 만약 다른 모듈의 데이터가 필요하다면, 반드시 해당 모듈이 노출한 인터페이스(Public API)를 통해 요청해야 합니다. 이러한 제약은 데이터 모델의 결합도를 낮추어, 훗날 특정 모듈을 마이크로서비스로 분리해야 할 때 데이터 이관 작업의 난이도를 획기적으로 낮춰줍니다.

![](/assets/img/2026-01-23-06-54-54.png)

### Transactional Consistency

공유 데이터베이스의 가장 큰 축복은 **ACID 트랜잭션**입니다. 여러 모듈에 걸친 작업을 수행할 때, 복잡한 2PC(Two-Phase Commit)나 Saga 패턴 없이도 데이터 정합성을 완벽하게 보장할 수 있습니다. 하지만 이는 양날의 검입니다.

편리함에 취해 여러 모듈의 로직을 하나의 거대한 트랜잭션으로 묶어버리면, 특정 모듈의 지연이 전체 시스템의 데이터베이스 커넥션 고갈로 이어질 수 있습니다. 따라서 모듈형 모노리스에서도 '트랜잭션의 범위'는 최대한 좁게 유지해야 합니다. 이상적인 구조는 주 도메인 로직은 로컬 트랜잭션으로 즉시 처리하고, 부수적인 작업(e.g., 알림 발송, 통계 업데이트)은 도메인 이벤트를 통해 비동기적으로 처리하여 트랜잭션의 전염을 막는 것입니다.

### Cross-Module Joins: The Anti-pattern

엔지니어들이 가장 자주 저지르는 유혹은 성능을 이유로 수행하는 **교차 모듈 조인(Cross-Module Join)**입니다. "API를 두 번 호출하는 것보다 한 번의 SQL 조인이 훨씬 빠르다"는 논리는 단기적으로는 맞지만, 아키텍처 관점에서는 치명적입니다.

교차 모듈 조인은 모듈 간의 물리적 결합을 고착화합니다. 특정 모듈의 테이블 구조가 변경되면 이를 조인하던 다른 모든 모듈의 쿼리가 깨지게 됩니다. 이는 캡슐화를 정면으로 위반하는 행위입니다.

| 구분            | 교차 모듈 조인 (Anti-pattern)        | API 기반 데이터 합성 (Recommended)              |
| :-------------- | :----------------------------------- | :---------------------------------------------- |
| **결합도**      | 매우 높음 (스키마 변경 시 연쇄 파손) | 낮음 (인터페이스 뒤로 숨겨짐)                   |
| **성능**        | 단일 쿼리로 최적화 가능              | 네트워크/애플리케이션 오버헤드 발생             |
| **분리 가능성** | MSA 전환 시 쿼리 전체 재작성 필요    | 서비스 분리 시 엔드포인트만 변경                |
| **권장 상황**   | **절대 금지** (기술 부채의 근원)     | 성능 이슈 발생 시 별도의 Query Model(CQRS) 구축 |

```java
// 1. Anti-pattern: DB Join을 통한 조회 (X)
// SELECT o.*, p.name FROM orders o JOIN products p ON o.product_id = p.id (X)

// 2. Recommended: ID 기반의 애플리케이션 서비스 결합 (O)
package com.example.order.application;

import com.example.catalog.CatalogClient; // Catalog 모듈의 API
import lombok.RequiredArgsConstructor;

@RequiredArgsConstructor
public class OrderQueryService {
    private final OrderRepository orderRepository;
    private final CatalogClient catalogClient;

    public OrderResponse getOrderDetails(String orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        
        // 타 모듈의 데이터는 DB 조인이 아닌 API(혹은 직접 참조)를 통해 가져옴
        ProductDto product = catalogClient.getProductById(order.getProductId());
        
        return new OrderResponse(order, product);
    }
}
```

### Distributed Tracing in Monolith

아이러니하게도 모듈형 모노리스에서도 **분산 추적(Distributed Tracing)** 기술이 필요합니다. 요청이 여러 모듈의 경계를 넘나들며 처리될 때, 어느 모듈에서 병목이 발생하는지, 어떤 모듈의 로직이 실패의 근원인지 파악하기 위해서입니다.

각 모듈 호출마다 고유한 `Trace ID`를 전달하고 로그에 남기는 습관은 시스템의 가시성을 극대화합니다. 이는 단순히 모니터링을 넘어, 향후 시스템이 비대해져 실제 마이크로서비스로 분할될 때 서비스 간의 통신 흐름을 미리 파악할 수 있는 귀중한 지도가 됩니다.

> **데이터는 모듈의 사생활이다**
> 
> 모듈의 소스 코드는 공유될 수 있어도, 그 모듈이 관리하는 데이터의 '물리적 구조'는 철저히 비밀로 유지되어야 합니다. 
> 
> 데이터베이스 테이블을 공유하는 것은 마치 다른 사람의 일기장을 허락 없이 읽는 것과 같습니다. 소통은 오직 인터페이스라는 대화를 통해서만 이루어져야 합니다.
{: .prompt-tip }

## Summary

Modular Monolithic Architecture는 **'운영의 단순함'**과 **'코드의 건전성'** 사이에서 최적의 균형을 찾는 전략적 선택입니다. 우리는 모듈을 통해 도메인을 격리하고(Module), 단일 배포를 통해 운영 비용을 절감하며(Monolith), 명확한 인터페이스와 아키텍처 가드레일을 통해 시스템의 부식을 막습니다. 특히 데이터베이스 스키마 수준의 논리적 격리를 실천할 때, 모노리스는 단순한 코드 뭉치를 넘어 언제든 마이크로서비스로 진화할 수 있는 유연한 생태계가 됩니다.

## References

* [[Shopify] Deconstructing the Monolith](https://engineering.shopify.com/blogs/engineering/deconstructing-the-monolith)
* [[Spring] Spring Modulith Project](https://spring.io/projects/spring-modulith/)
* [[ArchUnit] Unit test your Java architecture](https://www.archunit.org/)
* [[Milan Jovanović] Modular Monolith Data Isolation](https://www.milanjovanovic.tech/blog/modular-monolith-data-isolation)