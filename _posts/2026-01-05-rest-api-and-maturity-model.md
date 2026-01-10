---
layout: post
title: "[Web] REST API와 성숙도 모델"
date: 2026-01-05 23:28 +0900
categories:
- Computer Science
- Network
tags:
- Web
- REST
- REST API
- RMM
image:
    path: /assets/img/2026-01-06-00-14-14.png
---

도시를 설계할 때 도로의 폭, 건물의 용도 제한, 상하수도의 위치를 규정하는 법규는 창의성을 제한하는 것처럼 보입니다. 그러나 이러한 제약 덕분에 수백만 명의 시민이 예측 가능한 방식으로 상호작용하며 거대한 유토피아를 유지합니다. 소프트웨어 아키텍처의 세계에서 REST(Representational State Transfer)는 바로 이 '도시 계획 법규'와 같습니다. 

많은 개발자가 REST를 단순히 'JSON을 주고받는 HTTP API'로 오해하곤 합니다. 하지만 로이 필딩(Roy Fielding)이 그의 논문에서 정의한 REST는 단순히 데이터 형식을 규정하는 것이 아니라, 전 지구적 규모의 분산 시스템이 어떻게 수십 년간 붕괴하지 않고 진화할 수 있는지에 대한 철학적 해답을 제시합니다.

## 🧩 REST API

로이 필딩의 논문에 따르면, REST는 분산 하이퍼미디어 시스템을 위한 아키텍처 스타일입니다. 이는 특정 기술이나 프로토콜이 아니라, 시스템의 각 구성 요소가 지켜야 할 '제약 조건들의 집합'입니다. 이 제약들을 하나씩 준수할 때마다 시스템은 독보적인 가시성(Visibility), 신뢰성(Reliability), 그리고 확장성(Scalability)을 얻게 됩니다.

### Resource

REST의 가장 근본적인 단위는 리소스(Resource)입니다. 리소스는 이름이 붙여질 수 있는 모든 정보의 추상화입니다. 여기서 핵심은 리소스가 특정 시점의 데이터 값(Value)이 아니라, 그 개념적인 '매핑' 자체라는 점입니다.

예를 들어, `https://api.example.com/users/123/recent-orders`라는 URI는 '123번 사용자의 최근 주문 목록'이라는 개념적 대상을 가리킵니다. 실제 데이터는 시간이 흐름에 따라 변하겠지만, 리소스가 가리키는 대상의 본질은 변하지 않습니다. RESTful 설계의 첫 번째 실패 지점은 여기에 동사를 포함하는 것입니다. `/getUsers`나 `/createOrder`와 같은 경로는 리소스를 식별하는 것이 아니라 행위를 기술합니다. 이는 객체 지향 프로토콜로의 퇴행을 의미하며, URI의 고유한 식별 능력을 파괴합니다.

### Representation

리소스 자체는 추상적이기 때문에 서버와 클라이언트가 직접 리소스를 주고받을 수는 없습니다. 대신 서버는 리소스의 특정 시점의 상태를 캡처한 '표현(Representation)'을 전달합니다. 

우리가 흔히 접하는 JSON이나 XML은 리소스의 표현 양식 중 하나일 뿐입니다. 동일한 '사용자 리소스'라도 클라이언트의 요청(`Accept` 헤더)에 따라 프로필 이미지가 포함된 HTML이 될 수도 있고, 기계 판독이 가능한 JSON이 될 수도 있습니다. 


> **표현(Representation)의 구성 요소**
> 
> 리소스의 상태를 담은 데이터와 그 데이터를 설명하는 메타데이터(Content-Type 등)의 결합체입니다. 로이 필딩은 하이퍼미디어를 통해 애플리케이션의 상태가 전이되는 방식을 중시했습니다.
{: .prompt-info }

![](/assets/img/2026-01-10-15-34-34.png)

### Statelessness

REST의 무상태성(Statelessness) 제약은 서버가 클라이언트의 이전 요청 문맥을 기억하지 않아야 함을 의미합니다. 모든 요청은 그 자체로 완결적이어야 하며, 서버가 해당 요청을 처리하는 데 필요한 모든 정보를 포함해야 합니다.

Amazon Builders' Library의 자료에 따르면, 분산 시스템에서 서버는 언제든 실패할 수 있는 소모품(Cattle)입니다. 만약 서버가 세션 상태를 메모리에 들고 있다면, 해당 서버가 장애로 인해 재시작되거나 로드 밸런서에 의해 다른 노드로 요청이 전달될 때 시스템은 마비됩니다. 무상태성은 서버의 자원을 점유하는 '고착성'을 제거함으로써 수평적 확장(Scale-out)을 극대화합니다. 이는 PlanetScale의 자료에서 언급하는 것처럼 서버의 스레드와 메모리 자원을 효율적으로 관리하여, 동시 접속자가 폭증하는 환경에서도 시스템의 회복 탄력성을 유지하는 핵심 기제가 됩니다.

### Uniform Interface

REST를 다른 네트워크 아키텍처 스타일과 차별화하는 가장 강력한 제약은 '일관된 인터페이스(Uniform Interface)'입니다. 이는 시스템 구성 요소 간의 상호작용을 단순화하고 디커플링을 완성합니다. 로이 필딩은 이를 위해 네 가지 세부 제약을 제시했습니다.

1.  **리소스 식별**: URI를 통해 리소스를 명확히 구분합니다.
2.  **표현을 통한 리소스 조작**: 클라이언트가 리소스를 수정할 때, 리소스 자체가 아닌 그 표현을 서버에 전달하여 의도를 명시합니다.
3.  **자기 서술적 메시지(Self-descriptive messages)**: 메시지만 보고도 이를 어떻게 처리할지 알 수 있어야 합니다. (예: `Content-Type` 헤더가 반드시 존재해야 함)
4.  **HATEOAS(Hypermedia As The Engine Of Application State)**: 애플리케이션의 다음 상태로 가기 위한 경로가 응답 본문의 하이퍼링크를 통해 동적으로 제공되어야 합니다.

대부분의 현대 API는 1, 2번은 잘 지키지만 3, 4번에서 무너집니다. 특히 HATEOAS가 결여된 API는 클라이언트가 서버의 URI 구조를 미리 알고 있어야 하므로, 서버의 인터페이스가 변경될 때 클라이언트 코드도 함께 수정되어야 하는 강한 결합(Tight Coupling) 문제를 발생시킵니다. 

```http
# GET /users/123/profile HTTP/1.1
# Host: api.example.com
# Accept: application/hal+json

HTTP/1.1 200 OK
Content-Type: application/hal+json; charset=utf-8
Cache-Control: public, max-age=3600
Vary: Accept-Encoding

{
    "id": 123,
    "name": "Andrej Karpathy",
    "email": "andrej@example.com",
    "role": "Engineer",
    "_links": {
        "self": { "href": "/users/123/profile" },
        "orders": { "href": "/users/123/orders" },
        "update": { "href": "/users/123/profile", "method": "PATCH" }
    }
}
```

## 🧩 Maturity Model

로이 필딩의 이상은 높았지만, 현실의 엔지니어들은 당장 동작하는 코드를 짜야 했습니다. 마틴 파울러가 대중화시킨 '리처드슨 성숙도 모델(Richardson Maturity Model)'은 바로 이 지점, 즉 혼돈의 카오스에서 필딩의 유토피아로 향하는 계단을 정의합니다. 이 모델은 단순히 기술적 완성도를 측정하는 잣대가 아니라, 서비스가 복잡해짐에 따라 발생하는 엔드포인트 관리의 어려움을 어떻게 프로토콜의 힘으로 해결할 것인가에 대한 진화론적 관점을 제공합니다.

### Level 0 to 1: 리소스의 발견과 식별

성숙도 모델의 가장 밑바닥인 레벨 0은 'POX(Plain Old XML)의 늪'이라 불립니다. 이 단계에서 HTTP는 단순히 데이터를 실어 나르는 전송용 터널(Tunneling)에 불과합니다. 단 하나의 엔드포인트(예: `/api/service`)로 모든 요청을 보내고, 어떤 동작을 할지는 요청 본문(Body)에 적힌 함수 이름에 의존합니다. 이는 전형적인 RPC(Remote Procedure Call) 스타일로, 시스템이 커질수록 클라이언트는 서버 내부의 함수 이름을 모두 외워야 하는 강한 결합에 빠지게 됩니다.

레벨 1로의 진화는 '리소스'라는 개념을 도입하며 시작됩니다. 이제 클라이언트는 거대한 서비스 하나와 대화하는 것이 아니라, 개별적으로 식별되는 리소스들과 대화합니다. `/users`, `/orders/1`과 같이 명사 형태의 경로가 나타나며, 복잡한 비즈니스 로직이 리소스라는 단위로 쪼개지기 시작합니다. 하지만 이 단계에서도 여전히 `POST` 메서드 하나로 조회, 생성, 삭제를 모두 처리하는 '동사적 사고'에서 완전히 벗어나지는 못합니다.

### Level 2: HTTP Verbs를 통한 규약의 완성

레벨 2는 현대의 대다수 '준수한' API들이 머무는 지점입니다. 여기서는 HTTP 프로토콜의 표준 메서드(GET, POST, PUT, DELETE)를 의미에 맞게 사용합니다. 이 단계가 중요한 이유는 단순히 '동사'를 '메서드'로 바꿨기 때문이 아니라, 프로토콜 수준에서 **안전성(Safe)**과 **멱등성(Idempotency)**이라는 강력한 약속을 활용할 수 있게 되기 때문입니다.

Amazon Builders' Library의 분석에 따르면, 분산 시스템에서 재시도(Retry)는 피할 수 없는 선택입니다. 이때 API가 레벨 2를 준수하여 `GET`이나 `PUT`과 같은 멱등 메서드를 올바르게 구현하고 있다면, 클라이언트는 네트워크 타임아웃이 발생했을 때 시스템의 부작용을 걱정하지 않고 안전하게 재시도를 수행할 수 있습니다. 반면, 모든 것을 `POST`로 처리하는 레벨 1 이하의 시스템에서는 재시도가 중복 결제나 중복 생성 같은 치명적인 데이터 오염으로 이어질 위험이 큽니다.

또한, 레벨 2는 HTTP 상태 코드를 통해 서버의 상태를 전달합니다. Microsoft의 API 가이드라인이 강조하듯, `200 OK`와 `201 Created`, 혹은 `404 Not Found`와 같은 표준 코드를 사용함으로써 클라이언트는 응답 본문을 파싱해 보지 않고도 요청의 성공 여부를 즉각 판단할 수 있는 가시성을 얻게 됩니다.

![](/assets/img/2026-01-10-15-35-44.png)

### Level 3: 하이퍼미디어를 통한 엔진으로서의 상태 제어 (HATEOAS)

성숙도의 정점인 레벨 3은 HATEOAS(Hypermedia As The Engine Of Application State)를 실현합니다. 이 단계에서 서버의 응답은 현재 리소스의 데이터뿐만 아니라, 클라이언트가 다음 단계로 수행할 수 있는 행동(Link) 정보를 포함합니다. 

이것은 혁명적인 변화입니다. 클라이언트는 더 이상 특정 비즈니스 흐름을 위해 어떤 URL로 요청을 보내야 할지 미리 알 필요가 없습니다. 마치 웹 브라우저에서 하이퍼링크를 타고 다음 페이지로 이동하듯, API가 제공하는 링크를 따라가기만 하면 됩니다. 이는 서버가 인터페이스나 경로를 변경하더라도 클라이언트 코드를 수정할 필요가 없게 만드는 '궁극의 디커플링'을 약속합니다. 하지만 이 약속에는 만만치 않은 대가가 따릅니다.

> **HATEOAS 응답의 구조적 차이**
> 
> 단순히 데이터를 반환하는 것을 넘어, 응답 본문에 `_links`나 `rel`과 같은 메타데이터를 포함시켜 애플리케이션의 상태 전이를 가이드합니다.
{: .prompt-tip }

```json
// Level 0: The Swamp of POX (RPC 스타일)
// POST /userService
{
    "function": "cancelOrder",
    "orderId": "ORD-99",
    "reason": "changed mind"
}

// --------------------------------------------------

// Level 3: HATEOAS (HAL 표준 적용)
// GET /orders/99
{
    "orderId": "ORD-99",
    "status": "AWAITING_PAYMENT",
    "amount": 150.00,
    "currency": "USD",
    "_links": {
        "self": { "href": "/orders/99" },
        "payment": { "href": "/orders/99/payment" },
        "cancel": { "href": "/orders/99/cancellation" }
    }
}
```

## HATEOAS Trade-off

로이 필딩은 REST의 정수가 하이퍼미디어(HATEOAS)에 있다고 단언했습니다. 그는 "만약 애플리케이션 상태의 엔진이 하이퍼미디어가 아니라면, 그것은 REST가 아니다"라고 선언할 정도로 이 제약 조건을 강조했습니다. 하지만 실제 산업 현장에서 레벨 3 성숙도를 달성한 API를 찾기란 보물찾기만큼이나 어렵습니다. 왜 이 '아키텍처적 유토피아'는 개발자들에게 축복인 동시에 저주가 되었을까요?

### 클라이언트 결합도 해제의 이론적 이상

HATEOAS의 가장 큰 축복은 클라이언트와 서버 사이의 **완전한 디커플링**입니다. 일반적인 API 설계에서 클라이언트는 "주문을 취소하려면 `POST /orders/{id}/cancel`로 요청을 보내야 한다"는 사실을 코드로 하드코딩합니다. 이는 서버가 비즈니스 로직이나 URL 구조를 변경할 때 클라이언트 앱의 배포가 강제됨을 의미합니다.

반면, 로이 필딩의 철학을 구현한 API에서 클라이언트는 오직 '진입점(Entry Point)'만을 알고 시작합니다. 서버는 응답 본문에 현재 상태에서 가능한 다음 행동들을 링크 형식으로 동봉합니다. 

마틴 파울러(Martin Fowler)가 설명한 리처드슨 성숙도 모델에 따르면, 이 방식은 서버가 클라이언트의 파괴적 변경 없이 인터페이스를 진화시킬 수 있는 자율성을 부여합니다. 예를 들어, '주문 취소' 기능이 특정 조건(배송 시작 전)에서만 가능하다면, 서버는 해당 조건이 충족될 때만 취소 링크를 응답에 포함합니다. 클라이언트는 링크의 존재 여부만 체크하면 될 뿐, 복잡한 취소 가능 여부 로직을 클라이언트 사이드에서 중복 구현할 필요가 없어집니다.

![](/assets/img/2026-01-10-15-36-10.png)

### 현실적 네트워크 오버헤드와 오버엔지니어링의 경계

이러한 우아한 이론에도 불구하고, HATEOAS는 현업에서 거센 저항에 부딪힙니다. 가장 큰 이유는 **성능과 복잡성의 트레이드오프**입니다.

1.  **네트워크 오버헤드**: 매 응답마다 수많은 링크 메타데이터를 포함하는 것은 페이로드 크기를 키웁니다. 특히 모바일 환경처럼 대역폭이 제한적인 상황에서 수십 개의 URL 링크는 유의미한 레이턴시 증가를 초래합니다.
2.  **클라이언트의 구현 난이도**: 하이퍼미디어를 제대로 활용하려면 클라이언트는 단순한 데이터 파서를 넘어 '내비게이터' 역할을 수행해야 합니다. 이는 클라이언트 코드의 복잡도를 높이며, 개발 생산성을 저해하는 요인이 됩니다.
3.  **문서화의 역설**: 링크를 동적으로 준다고 해서 문서화가 필요 없는 것이 아닙니다. 오히려 각 링크의 `rel`(Relation)이 의미하는 바를 설명하는 별도의 메타 데이터 정의가 필요해지며, 이는 Swagger(OpenAPI)와 같은 표준 도구들과 매끄럽게 결합되지 않는 경우가 많습니다.

결국 많은 엔지니어들은 "서버와 클라이언트를 내가 모두 통제할 수 있는 상황인데, 굳이 이런 복잡성을 감수해야 하는가?"라는 실용적 질문을 던지게 됩니다. 이것이 대부분의 서비스가 레벨 2에 안주하며, HATEOAS를 '선택받은 자들의 사치'로 치부하게 된 배경입니다.

> **HATEOAS를 어디에 한정하여 도입해야 할까?**
> 
> 모든 API에 HATEOAS를 적용하기보다, 비즈니스 흐름이 복잡하고 상태 전이가 잦은 핵심 도메인(예: 복잡한 결제 프로세스)에 한정하여 부분적으로 도입하는 것이 엔지니어링 측면에서 합리적일 수 있습니다.
{: .prompt-warning }

```json
// GET /api/v1/orders/ORD-2026-99
{
    "orderId": "ORD-2026-99",
    "status": "AWAITING_PAYMENT",
    "customer": "Andrej Karpathy",
    "items": [
        { "name": "GeForce RTX 5090", "quantity": 1, "price": 1999.00 }
    ],
    "totalAmount": 1999.00,
    "currency": "USD",
    "createdAt": "2026-01-10T15:30:00Z",

    "_links": {
        "self": { "href": "/api/v1/orders/ORD-2026-99" },
        "payment": { 
            "href": "/api/v1/payments/pay",
            "method": "POST",
            "title": "결제하기" 
        },
        "cancel": { 
            "href": "/api/v1/orders/ORD-2026-99/cancellation",
            "method": "DELETE",
            "title": "주문 취소" 
        },
        "help": { "href": "https://docs.example.com/api/orders/status" }
    }
}
```

## Idempotency Strategy

성숙도 모델 레벨 2에서 HTTP 메서드를 규약에 맞게 사용하는 것은 시작에 불과합니다. 실제 분산 시스템의 전장에서 가장 다루기 까다로운 주제는 "네트워크는 언제나 실패할 수 있다"는 가정하에 시스템의 정합성을 유지하는 것입니다. 여기서 등장하는 개념이 바로 멱등성(Idempotency)입니다. 멱등성은 동일한 요청을 여러 번 수행하더라도 결과가 처음 한 번 수행했을 때와 같아야 함을 의미하며, 이는 단순한 아키텍처 스타일을 넘어 시스템의 생존 전략으로 기능합니다.

### PUT/DELETE와 Idempotency-Key 전략

HTTP 명세에 따르면 `GET`, `PUT`, `DELETE`는 본래 멱등합니다. 하지만 현실의 비즈니스 로직은 명세만큼 단순하지 않습니다. 예를 들어, `PUT /users/1` 요청이 네트워크 타임아웃으로 실패했을 때, 클라이언트는 이 요청이 서버에 도달해 처리가 끝났는지, 아니면 도달조차 못 했는지 알 길이 없습니다. 

Amazon Builders' Library의 설명에 따르면, 타임아웃이나 실패가 발생했다고 해서 부작용(Side-effects)이 일어나지 않았음을 보장하지는 않습니다. 만약 API가 멱등하게 설계되지 않았다면, 클라이언트의 안전한 재시도는 불가능해집니다. 이를 해결하기 위해 Stripe와 같은 선도적인 테크 기업들은 `Idempotency-Key`라는 커스텀 헤더를 실무 표준으로 정착시켰습니다.

클라이언트는 요청 시 고유한 키(UUID 등)를 생성하여 보냅니다. 서버는 이 키를 데이터베이스에 저장하고, 동일한 키로 요청이 다시 들어오면 실제 로직을 실행하는 대신 이전에 저장해두었던 응답을 그대로 반환합니다. Amazon EC2의 `RunInstances` API가 제공하는 토큰 기반 메커니즘 역시 이와 동일한 철학을 공유하며, 리소스 생성 작업에서도 안전한 재시도를 보장합니다.

### 분산 시스템에서의 중복 요청 처리 메커니즘

분산 아키텍처에서 멱등성이 결여된 재시도는 재앙을 초래합니다. Amazon의 분석에 따르면, 재시도는 본질적으로 '이기적인(Selfish)' 행위입니다. 클라이언트는 자신의 성공 가능성을 높이기 위해 서버의 자원을 더 많이 소모하도록 요구하기 때문입니다.

더 심각한 것은 재시도의 '증폭 현상'입니다. 5단계로 이뤄진 마이크로서비스 호출 체인에서 각 단계가 독립적으로 3번씩 재시도를 수행한다면, 하위 데이터베이스가 받는 부하량은 원래의 243배까지 폭증할 수 있습니다. 이러한 상황에서 멱등성이 담보되지 않는다면, 중복 데이터 생성과 부하 폭증이 맞물려 시스템은 거대한 '데스 스파이럴(Death Spiral)'에 빠지게 됩니다. 

따라서 멱등성 설계는 단순히 '중복 결제 방지' 수준을 넘어, 지수 백오프(Exponential Backoff)와 지터(Jitter)를 적용한 재시도 전략이 시스템을 파괴하지 않도록 만드는 최후의 보루입니다. 엔지니어는 표준 HTTP 메서드를 사용하는 것을 넘어, 비즈니스 에러와 프로토콜 실패가 엉키는 지점에서 어떻게 상태를 원자적(Atomic)으로 관리할지 고민해야 합니다.

> **멱등성 설계의 골든 룰**
> 
> 읽기 전용 API는 자연스럽게 멱등하지만, 리소스 생성(POST)이나 복잡한 상태 전이는 명시적인 토큰 메커니즘이 필요합니다. 멱등성은 클라이언트가 '안심하고 다시 시도'할 수 있는 권리를 부여합니다.
{: .prompt-tip }

```java
// [CODE_ID_03: Stripe/Amazon 스타일의 멱등성 처리 로직 (Java)]

@PostMapping("/v1/orders")
public ResponseEntity<OrderResponse> createOrder(
    @RequestHeader(value = "Idempotency-Key", required = false) String idempotencyKey,
    @RequestBody OrderRequest request
) {
    // 1. 멱등성 키가 없는 경우: 일반적인 비즈니스 로직 수행
    if (idempotencyKey == null || idempotencyKey.isBlank()) {
        return ResponseEntity.ok(orderService.process(request));
    }

    // 2. 저장소(Redis 등)에서 기존 처리 내역 확인 (Atomic Look-up)
    // Amazon Builders' Library의 권고에 따라, 이미 완료된 요청은 중복 실행 없이 즉시 반환함
    IdempotencyRecord cachedRecord = idempotencyRepository.findByKey(idempotencyKey);
    
    if (cachedRecord != null) {
        if (cachedRecord.isProcessing()) {
            // 동일 키로 현재 처리 중인 요청이 있는 경우 409 Conflict 반환
            return ResponseEntity.status(HttpStatus.CONFLICT).build();
        }
        // 저장된 응답 객체를 반환 (Payload와 상태 코드 복원)
        return ResponseEntity.status(cachedRecord.getStatusCode())
                             .body(cachedRecord.getResponsePayload());
    }

    // 3. 멱등성 레코드 생성 (처리 중 상태로 표시하여 Race Condition 방지)
    idempotencyRepository.save(new IdempotencyRecord(idempotencyKey, Status.PROCESSING));

    try {
        // 4. 실제 비즈니스 로직 수행 (결제, 리소스 생성 등 Side-effects 발생 지점)
        OrderResponse response = orderService.process(request);

        // 5. 처리 결과 및 응답 상태 저장 (24시간 등 TTL 설정 권장)
        idempotencyRepository.updateResult(idempotencyKey, response, HttpStatus.OK);

        return ResponseEntity.ok(response);

    } catch (Exception e) {
        // 6. 예외 발생 시 멱등성 레코드 삭제 또는 실패 상태 기록
        // 클라이언트가 이후에 동일한 키로 안전하게 재시도할 수 있도록 보장함
        idempotencyRepository.deleteByKey(idempotencyKey);
        throw e;
    }
}
```

## HTTP Semantics Pollution

우리는 종종 모든 응답이 `200 OK`로 돌아오지만, 그 속을 열어보면 `{"success": false, "message": "Balance insufficient"}`와 같은 메시지가 담긴 API를 마주합니다. 이것은 HTTP라는 정교한 언어를 단지 '데이터 전송 봉투'로 전락시키는 행위입니다. 로이 필딩이 강조한 '자기 서술적 메시지(Self-descriptive messages)'의 관점에서 볼 때, 이러한 설계는 캐시 프록시나 로드 밸런서와 같은 네트워크 중간 계층(Intermediaries)의 눈을 가리는 행위와 같습니다.

### 200 OK 뒤에 숨겨진 비즈니스 에러의 모순

HTTP 상태 코드는 단순히 성공과 실패를 알리는 신호등이 아닙니다. 그것은 전 지구적 네트워크가 요청의 성격을 이해하고 처리하는 '공용어'입니다. 

Amazon Builders' Library의 설명에 따르면, HTTP는 클라이언트 에러(4xx)와 서버 에러(5xx)를 명확히 구분합니다. 클라이언트 에러는 동일한 요청을 재시도해도 성공할 가능성이 없음을 나타내며, 서버 에러는 일시적인 장애일 수 있으므로 나중에 다시 시도하면 성공할 수 있음을 의미합니다. 만약 모든 응답을 `200 OK`로 통일해버린다면, 표준적인 네트워크 장비나 라이브러리는 해당 요청이 재시도 가능한지, 아니면 즉시 중단해야 하는지 판단할 수 없게 됩니다. 이는 시스템 전반의 가시성과 복구 능력을 크게 저해합니다.

### 프로토콜 에러와 애플리케이션 에러의 정교한 분리 설계

진정으로 성숙한 API는 프로토콜 수준의 실패와 애플리케이션 수준의 비즈니스 실패를 우아하게 격리합니다. Microsoft의 API 가이드라인은 상태 코드를 의미론적으로 정확하게 사용할 것을 권고합니다.

예를 들어, 리소스를 찾지 못한 경우 `404 Not Found`를, 인증은 되었으나 권한이 없는 경우 `403 Forbidden`을 사용하는 식입니다. 여기서 한 걸음 더 나아가, 비즈니스 에러 코드는 HTTP 상태 코드가 아닌 응답 본문의 별도 필드(예: `errorCode`)로 정의해야 합니다. 이는 클라이언트 엔지니어가 "네트워크가 끊겼는가(Protocol Error)?"와 "내 잔액이 부족한가(Business Error)?"를 명확히 구분하여 처리할 수 있게 돕습니다.

> **에러 처리의 황금률**
> 
> HTTP 상태 코드는 '요청 처리의 결과'를 기술하고, 응답 본문의 에러 객체는 '실패의 구체적 이유'를 기술해야 합니다. 이 둘을 섞는 순간, API의 범용성은 무너집니다.
{: .prompt-info }

성숙도 모델의 계단을 오르는 것은 단순한 기술적 자존심의 문제가 아닙니다. 그것은 시스템의 예측 가능성을 높이고, Amazon이 언급한 '이기적인 재시도'가 시스템을 붕괴시키지 않도록 방어막을 구축하는 실천적인 엔지니어링입니다.

## Summary

REST API와 성숙도 모델은 단순히 '예쁜 URL'을 만드는 규칙이 아니라, 분산 시스템의 복잡성을 관리하고 클라이언트와 서버 간의 독립적 진화를 보장하기 위한 아키텍처적 장치입니다. 리소스 중심의 설계와 HTTP 메서드의 멱등성을 올바르게 활용할 때, 시스템은 네트워크 실패라는 가혹한 환경에서도 Amazon이 강조하는 회복 탄력성(Resilience)을 확보할 수 있습니다. 비록 HATEOAS와 같은 이상적 제약 조건이 현실적인 트레이드오프에 부딪히기도 하지만, 표준 규약을 준수하는 것은 결국 대규모 시스템을 유지보수하는 비용을 획기적으로 낮추는 가장 확실한 투자입니다.

## References

* [[Amazon] Timeouts, retries and backoff with jitter](https://aws.amazon.com/ko/builders-library/timeouts-retries-and-backoff-with-jitter/)
* [[Martin Fowler] Richardson Maturity Model](https://martinfowler.com/articles/richardsonMaturityModel.html)
* [[Roy Fielding] REST Dissertation (Chapter 5)](https://roy.gbiv.com/pubs/dissertation/top.htm)
* [[Stripe] Idempotent Requests (API Ref)](https://docs.stripe.com/api/idempotent_requests)
* [[Microsoft] REST API Guidelines](https://github.com/microsoft/api-guidelines/blob/vNext/Guidelines.md)