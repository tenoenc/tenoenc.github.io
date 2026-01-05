---
layout: post
title: "[Web] REST API와 성숙도 모델"
date: 2026-01-05 23:28 +0900
categories:
- Computer Science
- Network
tags:
- Web
- REST API
- RMM
image:
    path: /assets/img/2026-01-06-00-14-14.png
---

우리는 'RESTful'이라는 단어를 관용구처럼 사용하지만, 정작 로이 필딩이 정립한 아키텍처의 정수와는 거리가 먼 코드를 작성하곤 합니다. 단순히 JSON을 HTTP에 실어 나르는 것을 넘어, 웹이라는 분산 시스템이 가진 잠재력을 설계에 녹여낼 고민이 생략된 결과입니다. 진정한 REST API 설계는 엔드포인트를 깎는 작업이 아니라, 서버와 클라이언트가 독립적으로 진화할 수 있는 '무상태의 영토'를 구축하는 일입니다. 기술적 교조주의를 배제하고 웹의 본질적 제약 조건을 투영할 때, 비로소 세월을 견디는 견고한 인터페이스가 탄생합니다.

## 🧩 REST API

오늘날 많은 개발자가 HTTP를 통해 JSON 데이터를 주고받으면 그것을 'REST API'라고 부릅니다. 하지만 REST의 창시자인 로이 필딩(Roy Fielding)은 "하이퍼텍스트가 애플리케이션의 상태를 주도하지 않는다면 그것은 REST API가 아니다"라고 단호하게 말합니다. 우리가 기술 면접이나 실무에서 마주하는 REST는 사실 웹 아키텍처의 거대한 철학을 아주 얇게 베어낸 단면에 불과할지도 모릅니다.

### REST 아키텍처 개요

REST(Representational State Transfer)는 2000년 로이 필딩의 박사 학위 논문에서 처음 소개되었습니다. 이는 표준 프로토콜이 아니라, 분산 하이퍼미디어 시스템을 설계하기 위한 **아키텍처 스타일(Architectural Style)**의 집합입니다. 

REST의 본질은 웹이 이미 가지고 있는 잠재력을 최대한 활용하는 데 있습니다. 마틴 파울러가 설명한 리처드슨 성숙도 모델(RMM)의 Level 0인 'The Swamp of POX' 단계에서는 HTTP를 그저 원격 호출을 위한 단순한 전송 통로(Tunneling)로만 사용합니다. 하지만 진정한 REST는 웹의 구성 요소들이 서로의 구현 세부 사항을 모르더라도 독립적으로 진화할 수 있는 생태계를 지향합니다.

### REST의 6가지 제약 조건

로이 필딩은 REST가 성립하기 위해 반드시 지켜야 할 6가지 제약 조건을 정의했습니다. 이 조건들은 시스템의 확장성(Scalability), 유연성(Flexibility), 그리고 단순성(Simplicity)을 보장하기 위한 엔지니어링적 장치입니다.

1. **Client-Server**: 사용자 인터페이스와 데이터 저장소를 분리하여 각자가 독립적으로 진화하게 합니다.
2. **Stateless**: 각 요청은 서버가 이해하는 데 필요한 모든 정보를 포함해야 하며, 서버는 요청 간의 문맥(Context)을 저장하지 않습니다. 이는 서버의 가용성과 확장성을 극대화합니다.
3. **Cacheable**: 응답 데이터는 명시적으로 캐시 가능 여부를 포함해야 하며, 이를 통해 네트워크 효율을 높입니다.
4. **Layered System**: 클라이언트는 서버에 직접 연결되었는지, 중간 매체(Proxy, Gateway)를 거쳤는지 알 수 없어야 하며, 이를 통해 보안과 로드 밸런싱의 이점을 얻습니다.
5. **Code-on-Demand (Optional)**: 서버가 실행 가능한 코드(예: JavaScript)를 클라이언트에 전송하여 기능을 확장할 수 있습니다.
6. **Uniform Interface**: REST를 다른 아키텍처와 구분 짓는 가장 핵심적인 특징입니다.

특히 **Uniform Interface**는 자원의 식별, 표현을 통한 자원의 조작, **Self-Descriptive Messages**, 그리고 **HATEOAS**를 포함합니다. 여기서 'Self-Descriptive'란 메시지 자체가 자신을 어떻게 처리해야 하는지 설명해야 함을 뜻하며(예: Content-Type 헤더), 'HATEOAS'는 애플리케이션의 상태 전이가 응답 내의 하이퍼링크를 통해 동적으로 결정되어야 함을 의미합니다.

### Resource와 URI 설계 전략

REST API 설계의 시작은 '무엇(What)'을 다룰 것인지 정의하는 것, 즉 **자원(Resource)**의 식별입니다. 모든 자원은 고유한 URI를 통해 식별되어야 합니다. 

URI를 설계할 때 주의해야 할 점은 계층 구조를 명확히 하고 명사형 명칭을 사용하는 것입니다. 예를 들어 사용자의 프로필 정보를 가져오는 경로는 `/users/{id}/profile`과 같이 설계합니다. 이때 Microsoft API Guidelines에서는 자원의 컬렉션은 복수형(`users`)을 사용할 것을 권장하며, 필터링이나 정렬과 같은 자원의 '상태 표현'은 쿼리 파라미터(`?sort=desc`)를 활용할 것을 조언합니다.

### HTTP Methods와 상태 코드의 시맨틱

REST API는 HTTP 표준 메서드를 통해 자원에 대한 행위를 규정합니다. 이때 가장 중요한 개념은 **Idempotency(멱등성)**와 **Safety(안전성)**입니다.

* **GET**: 자원을 조회하며, 여러 번 호출해도 서버 상태를 변화시키지 않는 안전한(Safe) 메서드입니다.
* **POST**: 새로운 자원을 생성하며, 호출할 때마다 새로운 결과가 생기므로 멱등하지 않습니다.
* **PUT/PATCH**: 자원을 수정합니다. PUT은 전체 교체, PATCH는 부분 수정을 의미하며, 일반적으로 PUT은 멱등성을 보장해야 합니다.
* **DELETE**: 자원을 삭제하며, 여러 번 호출해도 삭제된 상태가 유지되므로 멱등합니다.

> **상태 코드의 전략적 활용**
> 
> 클라이언트와의 원활한 소통을 위해 적절한 상태 코드를 반환하는 것은 필수입니다. 단순히 `200 OK`를 남발하기보다, 생성이 완료되었다면 `201 Created`를, 클라이언트의 잘못된 요청에는 `400 Bad Request`를, 그리고 더 이상 지원하지 않는 API 경로에 대해서는 Microsoft 가이드라인에 따라 적절한 만료 정보를 포함하여 응답해야 합니다.
{: .prompt-tip }

```json
{
    "id": 123,
    "name": "Teno",
    "email": "teno@example.com",
    "links": [
        {
            "rel": "self",
            "href": "https://api.tenolab.com/users/123",
            "method": "GET"
        },
        {
            "rel": "update",
            "href": "https://api.tenolab.com/users/123",
            "method": "PATCH"
        },
        {
            "rel": "delete",
            "href": "https://api.tenolab.com/users/123",
            "method": "DELETE"
        }
    ]
}
```

결국 REST는 단순히 데이터를 전송하는 기술이 아닙니다. 웹의 설계 원칙을 존중하며, 서버와 클라이언트가 서로의 변화에 민감하게 반응하지 않아도 지속 가능한 시스템을 만드는 아키텍처 철학입니다.

## 🧩 Richardson Maturity Model (RMM)

우리가 설계한 API가 얼마나 'RESTful'한지를 평가할 때, 가장 널리 인용되는 지표는 레너드 리처드슨(Leonard Richardson)이 제안한 **리처드슨 성숙도 모델(RMM)**입니다. 마틴 파울러에 의해 대중화된 이 모델은 REST라는 이상향에 도달하기 위한 4단계(Level 0 ~ 3) 로드맵을 제시합니다. 중요한 점은 각 단계가 단순히 기술적 난이도를 의미하는 것이 아니라, **웹의 핵심 기술(URI, HTTP, Hypermedia)을 얼마나 깊이 있게 활용하느냐**를 보여준다는 것입니다.

### 성숙도 모델의 정의와 목적

성숙도 모델은 서비스의 품질을 줄 세우기 위한 도구가 아닙니다. 서버와 클라이언트 사이의 결합도를 낮추고, 웹이라는 거대한 분산 시스템 안에서 서비스가 얼마나 독립적으로 생존하고 진화할 수 있는지를 측정하는 지표입니다. 각 단계를 거칠 때마다 서비스는 더 높은 유연성과 범용성을 획득하게 됩니다.

### Level 0 & 1: 엔드포인트의 분리

**Level 0. The Swamp of POX**

가장 낮은 단계인 Level 0은 HTTP를 그저 데이터 전송을 위한 '터널'로만 사용합니다. 하나의 엔드포인트(예: `/api/service`)에 모든 요청을 던지고, 어떤 행위를 할지는 메시지 바디에 담긴 XML이나 JSON 명세에 의존합니다. 이는 웹의 아키텍처를 전혀 활용하지 않는 RPC(Remote Procedure Call) 방식의 전형입니다.

**Level 1. Resources**

Level 1에 들어서면 비로소 **자원(Resource)**이라는 개념이 등장합니다. 모든 요청을 하나의 주소로 보내는 대신, `http://api.tenolab.com/books/1`과 같이 개별 자원에 고유한 URI를 부여합니다. 자원을 식별하기 시작했다는 점에서 큰 진전이지만, 여전히 행위에 대한 구분은 모호하며 주로 POST 메서드 하나로 모든 처리를 수행하는 경우가 많습니다.

### Level 2. 표준 시맨틱의 적용

대부분의 성숙한 기업용 API가 머물고 있는 지점이 바로 Level 2입니다. 이 단계의 핵심은 **HTTP Verbs(Methods)**와 **상태 코드(Status Codes)**를 표준 명세에 맞게 사용하는 것입니다.

조회는 `GET`, 생성은 `POST`, 수정은 `PUT/PATCH`를 사용하며, 서버는 작업 결과에 따라 `201 Created`나 `404 Not Found` 같은 명확한 상태 코드를 반환합니다. 이를 통해 API는 **멱등성(Idempotency)**과 **안전성(Safety)**이라는 강력한 규약을 얻게 되며, 중간 매체(캐시 서버, 프록시)가 이 규약을 바탕으로 효율적인 최적화를 수행할 수 있게 됩니다.

### Level 3. 하이퍼미디어 컨트롤 (HATEOAS)

리처드슨 모델의 정점은 **HATEOAS(Hypermedia As The Engine Of Application State)**를 실현하는 Level 3입니다. 이 단계에서 클라이언트는 서버가 보내준 응답 속에 포함된 하이퍼링크를 통해 '다음에 무엇을 할 수 있는지'를 발견합니다.

클라이언트는 서버의 URI 구조를 미리 다 알고 있을 필요가 없습니다. 마치 우리가 웹사이트를 서핑할 때 주소창에 다음 주소를 직접 치지 않고 링크를 클릭하며 이동하듯이, API 클라이언트도 응답에 담긴 링크를 따라 상태를 전이시킵니다. 이것이 로이 필딩이 말한 "진정한 REST"의 모습이며, 서버의 내부 구조가 바뀌어도 클라이언트 코드가 깨지지 않는 궁극적인 디커플링을 선사합니다.

```http
/* Level 2: HTTP Methods + Status Codes */
HTTP/1.1 200 OK
Content-Type: application/json

{
    "orderId": "ORD-101",
    "status": "SHIPPED",
    "item": "Mechanical Keyboard"
}

/* Level 3: Hypermedia Controls (HATEOAS) */
HTTP/1.1 200 OK
Content-Type: application/hal+json

{
    "orderId": "ORD-101",
    "status": "SHIPPED",
    "item": "Mechanical Keyboard",
    "_links": {
        "self": { "href": "/orders/101" },
        "cancel": { "href": "/orders/101/cancel" },
        "track": { "href": "/orders/101/tracking" }
    }
}
```

### [실습] 단계별 API 리팩토링 분석

서비스가 진화함에 따라 요청과 응답의 형태가 어떻게 달라지는지 목격하는 것은 매우 흥미로운 경험입니다. 단순한 RPC 형태의 메시지가 어떻게 의미론적인 REST 메시지로 변모하는지 분석해 봅니다.

```text
Level 0: POST /appointmentService (바디에 'findSlot' 명령 포함)
Level 1: GET /slots/123 (자원별 개별 URI 부여)
Level 2: POST /slots/123/book (성공 시 201 Created 반환)
Level 3: 응답에 <link rel="cancel" href="/slots/123/cancel" /> 포함
```

### 'True REST'의 딜레마

하지만 실무에서 Level 3(HATEOAS)를 완벽히 구현하는 경우는 드뭅니다. 여기에는 명확한 트레이드오프가 존재합니다.

1. **데이터 오버헤드**: 모든 응답에 링크 정보를 포함해야 하므로 메시지 크기가 커집니다.
2. **구현 복잡도**: 클라이언트 개발자는 링크 관계를 해석하는 추가 로직을 작성해야 하며, 서버는 매 요청마다 유효한 링크를 동적으로 생성해야 합니다.

> **현실적인 엔지니어의 선택**
> 
> 로이 필딩은 HATEOAS가 빠진 API를 REST라고 부르지 말라고 했지만, 실무에서는 **Level 2의 효율성**과 **Level 3의 유연성** 사이에서 끊임없이 타협합니다. 만약 API 사용자가 한정되어 있고 문서화가 잘 되어 있다면, 굳이 Level 3의 비용을 지불하지 않는 것이 실용적인 선택일 수 있습니다.
{: .prompt-info }

### Self-Describing의 오해

REST의 제약 조건 중 하나인 **Self-Descriptive(자습적 메시지)**는 단순히 응답이 JSON이라는 뜻이 아닙니다. 메시지 안에 담긴 `id`, `name` 같은 필드가 무엇을 의미하는지 클라이언트가 외부 지식 없이도 해석할 수 있어야 함을 뜻합니다. 

우리는 대개 `Content-Type: application/json`으로 충분하다고 생각하지만, 사실 이것만으로는 메시지의 의미를 온전히 설명하지 못합니다. 진정한 의미의 Self-Descriptive를 달성하려면 IANA에 미디어 타입을 등록하거나, 프로필 링크를 제공하여 메시지의 규약을 명시해야 합니다.

```json
{
    "_links": {
        "self": { "href": "/users/teno" },
        "curies": [{ "name": "teno", "href": "https://api.tenolab.com/docs/rels/{rel}", "templated": true }]
    },
    "username": "teno",
    "total_posts": 42,
    "_embedded": {
        "teno:latest_post": {
            "_links": { "self": { "href": "/posts/100" } },
            "title": "REST API Maturity Model"
        }
    }
}
```

## 'True REST'의 딜레마: Self-Descriptive와 HATEOAS

우리가 만드는 대부분의 API는 리처드슨 성숙도 모델의 Level 2에 머물러 있습니다. 로이 필딩의 관점에서 본다면, 이는 REST 아키텍처의 핵심 이점인 '독립적 진화'를 포기한 반쪽짜리 설계에 불과합니다. 특히 **Self-Descriptive**와 **HATEOAS**는 진정한 REST를 완성하는 퍼즐의 마지막 조각이자, 동시에 실무 엔지니어들이 가장 빈번하게 타협하는 지점이기도 합니다.

### 왜 'application/json'은 불충분한가? (Self-Descriptive)

REST의 제약 조건 중 하나인 **Self-Descriptive(자습적 메시지)**는 메시지 그 자체로 해석이 가능해야 함을 의미합니다. 하지만 우리가 흔히 사용하는 `Content-Type: application/json`은 데이터가 JSON 형식이라는 사실만 알려줄 뿐, 그 안에 담긴 `userId`, `account_balance` 같은 필드가 구체적으로 무엇을 의미하는지 설명하지 못합니다.

결국 클라이언트는 API 문서를 별도로 찾아보거나 서버 개발자에게 물어봐야 하며, 이는 서버와 클라이언트 사이의 강한 결합(Coupling)을 유발합니다. 이를 해결하기 위해 로이 필딩이 제안한 방식은 **IANA**에 새로운 미디어 타입을 등록하거나, 응답 헤더에 해당 메시지의 의미를 명세한 'Profile' 링크를 포함하는 것입니다.

### 하이퍼미디어의 역설 (HATEOAS)

**HATEOAS(Hypermedia As The Engine Of Application State)**는 애플리케이션의 상태 전이가 서버로부터 전달된 하이퍼링크에 의해 결정되어야 한다는 원칙입니다. 

이론적으로 HATEOAS를 도입하면 클라이언트는 서버의 URI 구조를 미리 알 필요가 없습니다. 서버가 리소스의 경로를 바꾸더라도 응답 내의 링크 정보만 업데이트하면 클라이언트는 코드를 수정하지 않고도 바뀐 경로를 따라갈 수 있습니다. 하지만 실제 개발 환경에서는 다음과 같은 역설이 발생합니다.

* **결합의 전이**: URI에 대한 결합은 사라지지만, 대신 링크의 의미를 나타내는 **'Relation(rel)'**에 대한 결합이 생깁니다. 클라이언트는 여전히 `rel="cancel"`이 결제를 취소하는 행위임을 미리 알고 있어야 합니다.
* **불필요한 트래픽**: 모든 응답에 방대한 링크 정보를 포함해야 하므로 페이로드 크기가 커지고 네트워크 비용이 증가합니다.

### 효율성 vs 유연성

우리는 이론적 순수성과 비즈니스 효율성 사이에서 선택해야 합니다. 대다수의 API가 Level 2에 머무는 이유는 **"문서화된 API 명세(Swagger, ReDoc)"**가 HATEOAS의 역할을 어느 정도 대신해주기 때문입니다.

1. **내부 시스템**: 프론트엔드와 백엔드가 긴밀하게 소통하는 사내 서비스라면 HATEOAS의 유연성보다 Level 2의 빠른 개발 속도와 작은 데이터 크기가 더 매력적입니다.
2. **공개 API**: 전 세계 수많은 개발자가 사용하는 오픈 API라면, 서버의 변화가 클라이언트에게 미치는 영향을 최소화하기 위해 Level 3의 제약 조건을 엄격히 준수하는 것이 장기적으로 유리합니다.

> **Pragmatic REST**
> 
> 모든 제약 조건을 맹목적으로 따르기보다, 도메인의 성격에 따라 결정해야 합니다. 진정한 REST를 지향한다면 **HAL(Hypertext Application Language)**이나 **JSON-LD** 같은 미디어 타입을 도입하여 데이터의 의미를 명확히 하는 것부터 시작하십시오. 그것만으로도 시스템의 유지보수성은 비약적으로 상승합니다.
{: .prompt-tip }

```json
/* HTTP Response Header */
// Link: <https://api.tenolab.com/docs/profiles/user>; rel="profile"

/* HTTP Response Body */
{
    "_links": {
        "self": { "href": "/users/123" },
        "profile": { "href": "https://api.tenolab.com/docs/profiles/user" }
    },
    "id": 123,
    "name": "Teno",
    "role": "ADMIN"
}
```

### [실습] 하이퍼미디어가 클라이언트 코드에 미치는 영향

HATEOAS가 적용된 API를 사용할 때 클라이언트의 코드는 '분기문' 중심에서 '탐색' 중심으로 변합니다. 서버의 상태에 따라 응답에서 특정 링크(`cancel`, `refund` 등)를 내려주거나 빼버림으로써, 클라이언트는 비즈니스 로직을 별도로 검증할 필요 없이 링크의 존재 유무만으로 UI를 제어할 수 있게 됩니다.

```java
// Spring HATEOAS Traverson 사용 예시
Traverson traverson = new Traverson(new URI("https://api.tenolab.com/"), MediaTypes.HAL_JSON);

// 1. 홈 엔드포인트에서 시작하여 'orders' 링크를 타고, 
// 2. 그 중 특정 주문의 'cancel' 링크를 찾아 DELETE 요청을 보냄
String cancelUrl = traverson.follow("orders", "my-order", "cancel").asLink().getHref();

restTemplate.delete(cancelUrl); 
// 서버에서 주문 상태가 '배송중'으로 바뀌어 'cancel' 링크를 내려주지 않으면, 
// 클라이언트는 자연스럽게 취소 기능을 실행할 수 없게 됨 (비즈니스 로직의 서버 중앙화)
```

## 인증과 트래픽 제어

REST API는 무상태성(Stateless)을 지향합니다. 하지만 역설적으로 이 무상태성은 보안 설계에 있어 큰 도전 과제를 안겨줍니다. 서버가 클라이언트의 상태를 기억하지 않는 상황에서, 어떻게 매 요청이 정당한 권한을 가진 사용자인지 검증하고, 동시에 폭주하는 트래픽으로부터 시스템을 보호할 수 있을까요? 시니어 엔지니어가 되기 위해 단순한 기능 구현을 넘어, 보안과 가용성 사이의 정교한 균형을 설계해야 합니다.

### OAuth2와 JWT를 활용한 무상태(Stateless) 인증 아키텍처

전통적인 세션 기반 인증은 서버 메모리에 상태를 저장하므로 분산 환경에서 확장성의 병목이 됩니다. 이를 해결하기 위해 현대적인 REST API는 **OAuth2** 프레임워크와 **JWT(JSON Web Token)**를 표준으로 채택합니다.

JWT는 그 자체로 사용자 정보와 권한을 포함하고 있는 '자가 수용적(Self-contained)' 토큰입니다. 서버는 별도의 DB 조회 없이 토큰의 서명(Signature)만 검증하여 요청의 유효성을 판단할 수 있습니다. 이는 REST의 무상태성을 완벽하게 충족하며 서버의 수평적 확장(Scale-out)을 용이하게 합니다.

> **토큰 탈취와 블랙리스트 딜레마**
> 
> JWT의 가장 큰 약점은 한 번 발급된 토큰은 만료 전까지 제어가 불가능하다는 점입니다. 이를 보완하기 위해 **Access Token**의 수명은 짧게 가져가고, **Refresh Token**을 통해 주기적으로 재발급받는 전략을 취합니다. 만약 특정 토큰을 강제로 무효화해야 한다면, Redis 같은 고속 저장소에 '블랙리스트'를 운영하는 하이브리드 방식을 고려해야 합니다.
{: .prompt-warning }

### Throttling과 Rate Limiting

아무리 잘 설계된 API라도 특정 클라이언트의 과도한 요청이나 디도스(DDoS) 공격 앞에서는 무력해질 수 있습니다. 시스템의 가용성을 보장하기 위해 우리는 **Rate Limiting(처리율 제한)** 메커니즘을 도입해야 합니다.

1. **Leaky Bucket / Token Bucket**: 일정한 속도로 요청을 처리하거나, 미리 할당된 토큰만큼만 요청을 허용하는 알고리즘입니다. 가장 널리 사용되며 시스템의 부하를 일정하게 유지합니다.
2. **Fixed/Sliding Window**: 특정 시간 단위(예: 1분) 내 요청 수를 제한합니다. 구현이 간단하지만 윈도우 경계에서 트래픽이 몰리는 문제를 주의해야 합니다.

이러한 트래픽 제어는 API 게이트웨이(Spring Cloud Gateway, Kong 등) 레벨에서 처리하는 것이 비즈니스 로직 보호와 성능 면에서 유리합니다.

### Circuit Breaker 패턴

분산 시스템에서 특정 마이크로서비스의 장애는 API 전체로 전이될 수 있습니다. 이때 필요한 것이 **서킷 브레이커(Circuit Breaker)**입니다. 외부 서비스의 응답 지연이나 에러율이 임계치를 넘어서면 즉시 연결을 차단(Open)하고, 미리 준비된 'Fallback' 응답을 반환하여 시스템 전체가 다운되는 것을 막습니다.

### [실습] API 게이트웨이를 통한 인증 및 속도 제한 설정

Spring Cloud Gateway와 Redis를 활용하여 실제 실무 환경에서 어떻게 트래픽을 제어하는지 구성해 봅니다.

```java
@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("user-service", r -> r.path("/api/users/**")
            .filters(f -> f.requestRateLimiter(config -> config
                .setRateLimiter(redisRateLimiter())
                .setKeyResolver(userKeyResolver())))
            .uri("lb://USER-SERVICE"))
        .build();
}

@Bean
public RedisRateLimiter redisRateLimiter() {
    // replenishRate: 초당 토큰 생성량
    // burstCapacity: 최대 수용량 (순간 트래픽 대응)
    return new RedisRateLimiter(10, 20);
}

@Bean
KeyResolver userKeyResolver() {
    // 헤더의 API-Key를 기준으로 제한 대상 식별
    return exchange -> Mono.just(exchange.getRequest().getHeaders().getFirst("X-API-KEY"));
}
```

```bash
# 짧은 시간에 여러 번 요청 보내기
$ for i in {1..25}; do curl -I -H "X-API-KEY: my-teno-key" https://api.tenolab.com/api/users/123; done

# 응답 예시
# HTTP/1.1 429 Too Many Requests
# X-RateLimit-Remaining: 0
# X-RateLimit-Burst-Capacity: 20
# X-RateLimit-Replenish-Rate: 10

# 429 응답은 클라이언트에게 "잠시 후 다시 시도하라"는 명확한 시맨틱을 전달합니다.
```

엔지니어에게 보안은 '벽'을 세우는 것이고, 가용성은 '길'을 관리하는 것입니다. 이 두 가지가 조화를 이룰 때 비로소 신뢰할 수 있는 RESTful API 서비스가 완성됩니다.

## 과도한 REST 집착

시니어 엔지니어의 역량은 '기술을 아는 것'이 아니라 '기술의 한계를 아는 것'에서 드러납니다. REST API가 현대 웹의 표준으로 자리 잡았지만, 모든 비즈니스 문제를 REST라는 틀에 억지로 끼워 맞추려는 시도는 때로 시스템의 복잡도를 높이고 성능을 저하시키는 독이 되기도 합니다.

### REST vs GraphQL vs gRPC

우리는 REST가 만능이 아님을 인정해야 합니다. 특히 통신이 발생하는 **맥락(Context)**에 따라 더 적합한 도메인 언어가 존재합니다.

1. **REST API**: 자원 중심의 설계로 범용성이 높고 캐싱에 유리합니다. 주로 **공개 API(Open API)**나 프론트엔드-백엔드 간의 일반적인 통신에 적합합니다.
2. **GraphQL**: 클라이언트가 필요한 데이터만 쿼리하여 가져옵니다. 모바일 환경처럼 네트워크 대역폭이 제한적이고, 화면마다 요구하는 데이터의 형태가 파편화된 서비스에서 빛을 발합니다.
3. **gRPC**: Protocol Buffers와 HTTP/2를 기반으로 한 고성능 RPC 프레임워크입니다. 페이로드 크기가 작고 직렬화 속도가 압도적이어서, **마이크로서비스 간 내부 통신(East-West Traffic)**에 최적화되어 있습니다.

### Over-fetching과 Under-fetching 문제

REST의 고정된 리소스 구조는 필연적으로 두 가지 비효율을 낳습니다.

* **Over-fetching**: 사용자 이름만 필요한데 서버는 프로필 사진, 주소, 작성 글 목록까지 포함된 거대한 JSON을 내려줍니다. 이는 불필요한 네트워크 트래픽과 클라이언트 측의 메모리 낭비를 유발합니다.
* **Under-fetching**: 화면을 구성하기 위해 여러 개의 엔드포인트를 순차적으로 호출해야 하는 현상입니다. (예: `/users` 호출 후 각 사용자의 `/posts`를 다시 호출). 이는 **N+1 요청 문제**로 이어져 사용자 경험을 저하시킵니다.

> **Pragmatic RPC**
> 
> 모든 것을 RESTful하게 만들기 위해 `/users/1/activate-account` 대신 `/users/1`에 PATCH 요청을 보내고 상태 값을 변경하는 식의 설계에 너무 집착하지 마세요. 때로는 **행위 중심의 RPC 스타일 엔드포인트** 하나가 수십 개의 복잡한 REST 제약 조건보다 훨씬 명확하고 유지보수하기 쉬울 수 있습니다.
{: .prompt-tip }

### 하이퍼미디어의 실무적 회의론

로이 필딩은 HATEOAS가 없는 API는 REST가 아니라고 했지만, 현실 세계의 프론트엔드 개발자들은 API 응답에 포함된 수많은 링크들을 해석(Parsing)하는 로직을 작성하는 것을 반기지 않습니다. 

진정한 REST를 추구하느라 클라이언트 코드의 복잡도를 높이고 있다면, 그것은 '엔지니어의 자기만족'을 위해 '개발자 경험(DX)'을 희생하고 있는 것은 아닌지 자문해봐야 합니다. 

```json
/* REST API (Under-fetching: 2번의 요청 필요) */
// 1. GET /users/1
{ "id": 1, "name": "Teno", "post_ids": [101, 102] }
// 2. GET /posts/101
{ "id": 101, "title": "REST is hard", "content": "..." }

/* GraphQL (단 한 번의 요청으로 필요한 데이터만 수집) */
// Query
query {
  user(id: 1) {
    name
    posts(first: 1) {
      title
    }
  }
}

// Response
{
  "data": {
    "user": {
      "name": "Teno",
      "posts": [{ "title": "REST is hard" }]
    }
  }
}
```

### 엔지니어링은 결국 트레이드오프의 예술

REST에 집착하는 엔지니어는 "이것이 RESTful한가?"를 묻지만, 시니어 엔지니어는 **"이 설계가 우리 팀의 생산성과 시스템의 응답 속도에 어떤 영향을 주는가?"**를 묻습니다. 

무상태성, 캐시 가능성, 계층화된 구조 등 REST가 주는 이점은 명확합니다. 하지만 실무에서는 도메인의 복잡도와 팀의 숙련도, 그리고 인프라 환경을 고려하여 **'적당히 RESTful한'** 지점에서 타협할 줄 아는 유연함이 필요합니다. 그것이 바로 기술적 교조주의에서 벗어나 실질적인 가치를 창출하는 길입니다.

## Summary

REST API를 설계한다는 것은 단순히 HTTP 엔드포인트를 만드는 행위가 아니라, **웹의 기본 원칙을 활용하여 시스템의 지속 가능성을 확보하는 설계 철학**을 실천하는 것입니다.

| 구분            | 핵심 내용                                   | 비고                                   |
| :-------------- | :------------------------------------------ | :------------------------------------- |
| **핵심 가치**   | 독립적 진화 (Decoupling)                    | 서버와 클라이언트의 병렬적 발전 보장   |
| **RMM Level 2** | 자원 식별(URI) + 표준 시맨틱(Method/Status) | 가장 현실적이고 널리 쓰이는 표준       |
| **RMM Level 3** | 하이퍼미디어 컨트롤 (HATEOAS)               | 진정한 REST의 완성, 높은 유연성 제공   |
| **보안 전략**   | 무상태성 인증 (Stateless Auth)              | OAuth2 & JWT를 통한 확장성 확보        |
| **설계 태도**   | Pragmatic Approach (실용주의)               | REST의 교조주의에서 벗어난 유연한 선택 |

### 최종 엔지니어링 체크포인트
- **Uniform Interface의 준수**: 자원은 명사로, 행위는 HTTP 메서드로 명확히 분리했는가?
- **Self-Descriptive의 실현**: 응답 메시지만 보고도 클라이언트가 데이터를 해석할 수 있는가? (Content-Type, Profile 활용)
- **가용성 설계**: Rate Limiting과 Circuit Breaker를 통해 외부 공격과 연쇄 장애에 대비했는가?
- **적정 기술의 선택**: 도메인의 특성에 따라 REST가 아닌 GraphQL이나 gRPC가 더 유리하지는 않은가?

REST는 만능 열쇠가 아닙니다. 하지만 우리가 웹의 제약 조건을 깊이 이해하고 이를 적재적소에 활용할 때, 비로소 세월의 흐름을 견디는 견고한 API 아키텍처를 구축할 수 있습니다.

## References

* **[Roy Fielding]** [Architectural Styles and the Design of Network-based Software Architectures](https://roy.gbiv.com/pubs/dissertation/rest_arch_style.htm)
* **[Roy Fielding]** [REST APIs must be hypertext-driven](https://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven)
* **[Martin Fowler]** [Richardson Maturity Model: steps toward the glory of REST](https://martinfowler.com/articles/richardsonMaturityModel.html)
* **[Microsoft]** [Microsoft REST API Guidelines - Deprecation & Versioning](https://github.com/microsoft/api-guidelines/blob/vNext/graph/Guidelines-deprecated.md)
* **[IANA]** [Official Assignments of Media Types](https://www.iana.org/assignments/media-types/media-types.xhtml)