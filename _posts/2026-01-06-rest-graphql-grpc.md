---
layout: post
title: "[Web] REST vs GraphQL vs gRPC"
date: 2026-01-06 00:35 +0900
categories:
- Computer Science
- Network
tags:
- Web
- REST
- GraphQL
- gRPC
image:
    path: /assets/img/2026-01-06-05-09-39.png
---

분산 시스템의 시대에서 서비스 간의 '대화'는 단순한 데이터 전송을 넘어 시스템의 생존성과 직결됩니다. 과거 하나의 거대한 성벽(Monolith) 안에서 함수 호출로 해결되던 로직들은 이제 네트워크라는 불확실한 바다를 건너야 하는 숙명을 안게 되었습니다. 우리는 더 이상 "어떻게 연결할 것인가"를 고민하지 않습니다. 대신 "어떤 언어로, 얼마나 효율적으로 소통하여 시스템의 복잡도를 제어할 것인가"를 질문해야 합니다. REST의 보편성, GraphQL의 유연성, 그리고 gRPC의 압도적인 성능 사이에서 최적의 균형점을 찾는 여정은 엔지니어가 시스템 아키텍처를 설계하는 데 있어 가장 짜릿하고도 신중해야 할 순간입니다.

## 🧩 REST

오늘날 웹의 근간을 이루는 REST(Representational State Transfer)는 자원을 이름으로 구분하여 그 상태를 주고받는 가장 보편적인 방식입니다. 로이 필딩이 정의한 이 아키텍처 스타일은 새로운 프로토콜을 만드는 것이 아니라, 웹이 이미 가지고 있는 HTTP라는 거대한 인프라를 가장 영리하게 활용하는 방법을 제시합니다.

### Resource-Oriented Architecture (ROA)

REST의 본질은 모든 것을 자원(Resource)으로 정의하고, 고유한 URI를 통해 이를 식별하는 데 있습니다. 서버는 자원의 상태를 JSON이나 XML 형태로 표현하여 전달하고, 클라이언트는 HTTP 표준 메서드(GET, POST, PUT, DELETE)를 통해 해당 자원에 대한 행위를 규정합니다.

이 방식의 가장 큰 강점은 웹 생태계의 기존 인프라를 온전히 누릴 수 있다는 점입니다. 무상태성(Stateless)을 전제로 하는 REST API는 각 요청이 독립적이기 때문에 서버의 수평적 확장(Scale-out)이 용이합니다. 또한, HTTP의 표준 캐싱 규약을 충실히 따르므로 CDN이나 프록시 서버 레벨에서 데이터를 효율적으로 캐싱하여 서버의 부하를 획기적으로 줄일 수 있습니다. 이러한 보편성은 전 세계 수많은 개발자와 시스템이 별도의 학습 비용 없이 소통할 수 있는 공통 언어를 제공합니다.

### 과도한 추상화의 대가: Over-fetching & Under-fetching

하지만 REST의 고정된 엔드포인트 구조는 현대의 복잡한 UI 요구사항과 충돌할 때 명확한 물리적 한계를 드러냅니다. 서버가 미리 정의해둔 자원의 형태를 그대로 받아야만 하기 때문입니다.

1. **Over-fetching**: 클라이언트는 사용자의 이름 하나만 필요하지만, 서버는 프로필 사진, 주소, 작성 글 목록 등이 포함된 거대한 JSON을 내려줍니다. 이는 불필요한 네트워크 트래픽을 유발하고 모바일 환경에서 대역폭을 낭비하는 원인이 됩니다.
2. **Under-fetching**: 화면 하나를 구성하기 위해 여러 엔드포인트를 호출해야 하는 현상입니다. 예를 들어 사용자의 정보를 가져온 뒤, 다시 그 사용자의 게시글 목록 엔드포인트를 호출해야 합니다. 이러한 N+1 요청 문제는 네트워크 왕복 시간(RTT)을 증가시켜 애플리케이션의 응답 속도를 저하시킵니다.

> **보편성의 가치**
> 
> GraphQL이나 gRPC가 REST의 한계를 지적하며 등장했지만, 여전히 대다수의 공개 API(Open API)가 REST를 선택하는 이유는 시스템의 가시성과 범용성 때문입니다. 브라우저 주소창에 URI를 입력하거나 curl 명령어 하나로 즉시 결과를 확인할 수 있는 단순함은, 복잡한 시스템의 장애를 진단하고 생태계를 확장하는 데 있어 기술적 성능보다 더 중요한 가치를 제공하기도 합니다.
{: .prompt-tip }

### [실습] RESTful API 설계의 실제

전형적인 자원 중심 설계에서 URI와 HTTP 메서드가 어떻게 상호작용하여 의미를 형성하는지 분석해 봅니다.

```text
# 자원 지향적 API 설계 예시
GET    /users/123          # ID가 123인 사용자 조회
POST   /users              # 새로운 사용자 생성
PATCH  /users/123          # 사용자 정보 일부 수정
DELETE /users/123          # 사용자 삭제

# 관계 표현
GET    /users/123/posts    # 특정 사용자의 게시글 목록 조회
```

```json
// 클라이언트는 'username'만 원하지만, 서버는 전체를 응답 (Over-fetching)
{
  "id": 123,
  "username": "Teno",
  "email": "tenoenc@gmail.com",
  "profile_image": "https://api.tenolab.com/assets/img/teno.png",
  "bio": "Senior Java Backend Developer",
  "created_at": "2026-01-01T12:00:00Z",
  "last_login": "2026-01-06T03:00:00Z",
  "settings": {
    "theme": "dark",
    "notifications": true
  }
}
```

위 예시에서 알 수 있듯, REST는 명확한 규약을 통해 시스템의 예측 가능성을 높이지만, 클라이언트의 요구사항이 파편화될수록 엔드포인트 관리의 복잡도가 증가하는 트레이드오프를 안고 있습니다.

## 🧩 GraphQL

REST가 자원의 위치(URL)에 집중했다면, GraphQL은 자원 간의 **관계(Relationship)**에 집중합니다. 모든 데이터를 하나의 거대한 그래프로 바라보고, 클라이언트가 그 그래프의 특정 노드와 엣지를 탐색하여 필요한 정보만 추출해가는 방식은 API 설계의 주도권을 서버에서 클라이언트로 옮겨놓는 패러다임의 전환을 불러왔습니다.

### Query Language for APIs

GraphQL의 핵심은 서버와 클라이언트 사이의 명확한 **타입 시스템(Type System)**에 있습니다. 서버는 사용 가능한 데이터의 구조를 스키마(Schema)로 정의하고, 클라이언트는 이 스키마를 바탕으로 자신이 원하는 응답의 형태를 쿼리로 작성합니다.

이 과정에서 빛을 발하는 것이 **Introspection** 기능입니다. 클라이언트는 서버에게 "너가 어떤 타입을 가지고 있고, 어떤 쿼리를 지원하니?"라고 직접 물어볼 수 있습니다. 이러한 자기 문서화 능력은 개발 도구와 결합하여, API 명세서를 따로 뒤적이지 않고도 실시간으로 필드를 탐색하고 타입을 검증하며 코드를 작성할 수 있는 압도적인 개발자 경험(DX)을 선사합니다.

### 데이터 그래프 모델링과 관계의 탐색

GraphQL을 단순히 '필드를 골라 담는 도구'로만 이해하는 것은 이 기술의 본질을 놓치는 것입니다. GraphQL은 비즈니스 도메인을 **상호 연결된 노드들의 집합**으로 정의합니다.

예를 들어, 한 사용자가 작성한 게시글들과 그 게시글에 달린 댓글들을 가져올 때, REST는 여러 번의 엔드포인트 호출이 필요하지만 GraphQL은 그래프를 따라가며(Traversing) 단 한 번의 쿼리로 모든 관계를 맺어줍니다. 이는 백엔드에서 복잡하게 얽힌 엔티티들을 클라이언트가 자신의 뷰(View) 구조에 맞춰 자유롭게 재구성할 수 있음을 의미합니다. PayPal의 사례처럼 수많은 마이크로서비스로 파편화된 데이터를 하나의 일관된 인터페이스로 묶어주는 '데이터 오케스트레이션' 계층으로서 GraphQL이 각광받는 이유가 바로 여기에 있습니다.

### 유연함의 이면: Resolver 최적화와 보안

하지만 클라이언트에게 부여된 이 강력한 자유에는 서버 엔지니어가 짊어져야 할 기술적 부채가 뒤따릅니다. 바로 **Resolver**의 효율성 문제입니다.

클라이언트가 중첩된 관계를 쿼리할 때, 서버 내부에서는 각 필드에 대응하는 함수(Resolver)들이 호출됩니다. 이때 아무런 최적화 없이 DB를 조회하면, 10개의 게시글에 대해 각각 작성자 정보를 가져오기 위해 10번의 추가 쿼리가 발생하는 **N+1 Query** 문제가 발생합니다. 이를 해결하기 위해 자바 생태계에서는 요청을 배치(Batch)로 모아 처리하는 DataLoader 패턴 등을 활용하여 데이터베이스 계층의 부하를 관리해야 합니다.

또한, 무상태(Stateless) HTTP 캐싱을 적극 활용하는 REST와 달리, GraphQL은 대개 POST 메서드 하나를 공유하므로 브라우저나 CDN 레벨의 캐싱이 까다롭습니다. 클라이언트 측에서 별도의 캐시 라이브러리를 운용해야 하거나, 서버에서 쿼리의 복잡도(Query Complexity)를 계산하여 악의적인 심층 쿼리로부터 시스템을 보호하는 추가적인 설계가 필요합니다.

> **BFF와 GraphQL의 시너지**
> 
> 모바일, 웹, 스마트 워치 등 다양한 클라이언트가 존재하는 환경에서 각 기기마다 최적화된 데이터를 제공하는 것은 매우 고통스러운 작업입니다. 이때 **BFF(Backend For Frontend)** 패턴을 적용하고 그 인터페이스로 GraphQL을 채택하면, 백엔드 마이크로서비스의 변경 없이도 클라이언트가 스스로 필요한 최적의 페이로드를 구성할 수 있는 유연성을 확보하게 됩니다.
{: .prompt-tip }

### [실습] Schema Definition Language (SDL) 분석

GraphQL 서버의 설계도인 SDL을 통해 데이터가 어떻게 타입화되고 관계를 맺는지 확인해 봅니다.

```graphql
type User {
  id: ID!
  username: String!
  posts: [Post]
}

type Post {
  id: ID!
  title: String!
  content: String
  author: User
  comments: [Comment]
}

type Comment {
  id: ID!
  body: String
  commenter: User
}

type Query {
  user(id: ID!): User
}
```

```graphql
# 클라이언트가 요청하는 쿼리 예시
query GetUserPostDetails($userId: ID!) {
  user(id: $userId) {
    username
    posts {
      title
      comments {
        body
        commenter {
          username
        }
      }
    }
  }
}
```

위 쿼리에서 알 수 있듯, 클라이언트는 자신이 정의한 구조를 정확히 그대로 돌려받습니다. 이는 서버와 클라이언트 사이의 '데이터 형태에 대한 협상' 비용을 0으로 수렴하게 만들며, 프론트엔드 개발자가 서버 API의 변경을 기다리지 않고도 독립적으로 UI를 진화시킬 수 있는 동력이 됩니다.

## 🧩 gRPC

성능이 곧 비용인 대규모 분산 시스템 환경에서, 텍스트 기반의 JSON과 REST 아키텍처는 때로 사치스러운 오버헤드가 됩니다. gRPC는 이러한 비효율을 걷어내고, 기계 간 통신(M2M)의 효율성을 극대화하기 위해 구글에서 설계한 고성능 RPC 프레임워크입니다. 단순히 '빠른 통신'을 넘어, 엄격한 타입 안정성과 인터페이스 정의 언어(IDL)를 통한 강력한 협업 모델을 지향합니다.

### Protocol Buffers와 HTTP/2의 결합

gRPC의 압도적인 성능은 **Protocol Buffers(Protobuf)**라는 바이너리 직렬화 포맷에서 기인합니다. 인간이 읽기 편한 JSON이 필드명을 매번 텍스트로 포함하며 용량을 차지하는 것과 달리, Protobuf는 필드 번호와 타입 정보를 결합한 태그-값(Tag-Value) 쌍의 바이너리 형태로 데이터를 인코딩합니다.

![](/assets/img/2026-01-06-05-12-52.png)

예를 들어, 숫자 데이터를 저장할 때 Protobuf는 **Varints**라 불리는 가변 길이 정수 인코딩 방식을 사용하여, 작은 숫자는 단 몇 바이트만으로도 표현할 수 있게 설계되었습니다. 이러한 고도의 압축은 네트워크 대역폭 소모를 획기적으로 줄여주며, 직렬화 및 역직렬화에 드는 CPU 연산 비용 역시 JSON 대비 압도적으로 낮습니다.

전송 계층에서는 **HTTP/2**를 채택하여 현대적인 네트워크 기술의 정수를 활용합니다. 단일 TCP 연결 내에서 여러 요청을 동시에 처리하는 멀티플렉싱(Multiplexing)은 물론, 바이너리 프레임 처리를 통해 텍스트 프로토콜이 가진 고질적인 성능 병목을 해결합니다. 특히 헤더 압축 기술은 반복되는 메타데이터의 크기를 최소화하여 실제 데이터 전송 효율을 극대화합니다.

### 강한 결합과 브라우저의 장벽

gRPC는 **IDL(Interface Definition Language)**인 .proto 파일을 통해 클라이언트와 서버 사이의 '엄격한 계약'을 맺습니다. 이 파일을 바탕으로 다양한 언어의 클라이언트 스텁(Stub)과 서버 코드가 자동으로 생성되므로, 개발자는 통신 로직이 아닌 비즈니스 로직에만 집중할 수 있습니다.

하지만 이러한 강한 결합은 장점인 동시에 제약이 되기도 합니다. 브라우저 환경에서는 아직 HTTP/2의 로우 레벨 프레임에 직접 접근할 수 있는 API가 부족하여, 네이티브 gRPC를 직접 호출하는 데 한계가 있습니다. 이를 위해 gRPC-Web이라는 프록시 계층을 두어야 하는 아키텍처적 복잡성이 발생합니다. 또한 바이너리 포맷의 특성상 별도의 도구 없이는 요청 내용을 눈으로 확인하거나 디버깅하기 어렵다는 점도 엔지니어가 감수해야 할 트레이드오프입니다.

> **내부 트래픽의 해방**
> 
> 마이크로서비스 간의 통신(East-West Traffic)에서 REST를 고집하는 것은 시스템에 보이지 않는 세금을 내는 것과 같습니다. 수백 개의 서비스가 서로 통신하는 복잡한 아키텍처라면, gRPC 도입을 통한 CPU 및 대역폭 절감은 단순한 수치 개선을 넘어 실제 클라우드 인프라 비용의 유의미한 감소로 이어집니다.
{: .prompt-tip }

### [실습] Protobuf 정의와 서비스 인터페이스 분석

gRPC 통신의 설계도인 .proto 파일을 통해 서비스가 어떻게 명세되는지 확인해 봅니다.

```protobuf
syntax = "proto3";

package tenolab.user;

option java_multiple_files = true;
option java_package = "com.tenolab.grpc.user";

// 회원 정보 메시지 정의
message UserProfile {
  int32 id = 1;
  string username = 2;
  string email = 3;
}

message UserRequest {
  int32 user_id = 1;
}

// 서비스 인터페이스 정의
service UserService {
  // Unary RPC: 단일 요청 및 단일 응답
  rpc GetUserProfile (UserRequest) returns (UserProfile);
  
  // Server Streaming: 한 번의 요청에 여러 응답 스트림
  rpc GetUserPosts (UserRequest) returns (stream PostResponse);
}

message PostResponse {
  string title = 1;
  string content = 2;
}
```

```java
public class UserServiceImpl extends UserServiceGrpc.UserServiceImplBase {

    @Override
    public void getUserProfile(UserRequest request, 
                               StreamObserver<UserProfile> responseObserver) {
        // 비즈니스 로직: 유저 조회 시뮬레이션
        UserProfile response = UserProfile.newBuilder()
                .setId(request.getUserId())
                .setUsername("Teno")
                .setEmail("teno@tenolab.com")
                .build();

        // 결과 반환 및 스트림 종료
        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }
}
```

위 명세에서 알 수 있듯, gRPC는 단방향(Unary) 요청뿐만 아니라 클라이언트 스트리밍, 서버 스트리밍, 그리고 양방향 스트리밍이라는 4가지 통신 모델을 네이티브하게 지원합니다. 이는 실시간 주가 데이터나 대용량 파일 전송처럼 지속적인 데이터 흐름이 필요한 서비스 구축 시, REST로는 흉내 내기 어려운 강력한 유연성을 제공합니다.

## 아키텍처 선택의 예술: Trade-off 분석

엔지니어링의 정수는 완벽한 기술을 찾는 것이 아니라, 현재의 비즈니스 맥락과 시스템 환경에서 최적의 **트레이드오프(Trade-off)**를 선택하는 데 있습니다. REST, GraphQL, gRPC 중 무엇이 더 우월한지를 묻는 것은 우문이며, 우리가 해결해야 할 병목의 위치가 어디인지를 먼저 파악하는 것이 현명한 엔지니어의 자세입니다.

### 도메인 맥락에 따른 기술 배치

우리가 구축하는 서비스의 성격에 따라 기술의 가치는 달라집니다.

1. **공개 API와 생태계 확장성**: 불특정 다수의 개발자가 사용하는 외부 공개용 API라면 고민할 것 없이 **REST**가 정답입니다. 범용적인 도구 지원과 낮은 진입 장벽, 그리고 HTTP 캐싱 인프라의 혜택은 기술적 성능보다 훨씬 강력한 가치를 제공합니다.

2. **파편화된 모바일 환경과 DX**: 프론트엔드 팀의 요구사항이 빈번하게 바뀌고, 네트워크 환경이 열악한 모바일 사용자가 주 고객이라면 **GraphQL**이 빛을 발합니다. 클라이언트가 직접 데이터의 형태를 정의하게 함으로써 백엔드 팀과의 커뮤니케이션 비용을 줄이고, 단 한 번의 요청으로 화면을 구성하는 '데이터 효율성'은 비즈니스 민첩성을 극대화합니다.

3. **고속 마이크로서비스 내부 통신**: 사용자에게 보이지 않는 백엔드 간 통신(East-West Traffic)은 성능과 비용의 영역입니다. 페이로드의 크기를 최소화하고 CPU 효율을 높여야 하는 상황에서는 **gRPC**가 압도적인 우위를 점합니다. 특히 강력한 타입 안정성은 수백 개의 서비스가 얽힌 환경에서 장애를 방지하는 든든한 방어선이 됩니다.

### 하이브리드 아키텍처 설계: BFF 패턴

현대의 정교한 시스템은 단일 기술에 집착하지 않고 각 기술의 장점을 조합하는 **하이브리드 전략**을 취합니다. 그 중심에는 **BFF(Backend For Frontend)** 패턴이 존재합니다.

이 구조에서 마이크로서비스들은 내부적으로 **gRPC**를 통해 고속으로 데이터를 주고받으며 성능을 확보합니다. 그리고 클라이언트와 마주하는 최전선(BFF)에서는 **GraphQL**이나 **REST**를 배치하여 각 플랫폼(Web, iOS, Android)에 최적화된 데이터를 서빙합니다. 특히 PayPal의 성공 사례처럼, 여러 마이크로서비스에서 흩어진 데이터를 GraphQL이 오케스트레이션하여 클라이언트에게 단일 뷰를 제공하는 방식은 현대적 백엔드 아키텍처의 정석으로 자리 잡고 있습니다.

> **기술적 교조주의 경계**
> 
> 로이 필딩의 REST 제약 조건을 완벽히 지키지 않았다고 해서, 혹은 gRPC의 스트리밍 기능을 쓰지 않는다고 해서 잘못된 설계는 아닙니다. 엔지니어는 기술의 '순수성'보다 '실효성'에 집중해야 합니다. 프로젝트 초기에 GraphQL의 복잡도가 부담스럽다면 REST로 시작하고, 이후 병목이 발생하는 지점에 선택적으로 gRPC나 GraphQL을 도입하는 유연함이 필요합니다.
{: .prompt-tip }

### [실습] 아키텍처 의사결정 매트릭스

팀 내에서 기술 스택을 결정할 때 참고할 수 있는 간단한 판단 기준을 코드로 명세해 봅니다.

```java
public enum ApiProtocol {
    REST, GRAPHQL, GRPC
}

public class ArchitectureConsultant {
    public ApiProtocol recommend(DomainContext context) {
        if (context.isPublicExternalAPI()) {
            return ApiProtocol.REST; // 보편성과 범용성 우선
        }
        
        if (context.hasComplexMobileUI() || context.isDataOrchestrationNeeded()) {
            return ApiProtocol.GRAPHQL; // 클라이언트 유연성 및 N+1 요청 해결
        }
        
        if (context.isInternalMicroservice() && context.isPerformanceCritical()) {
            return ApiProtocol.GRPC; // 성능 및 타입 안정성 최우선
        }
        
        return ApiProtocol.REST; // 기본값
    }
}
```

이러한 논리적 근거를 바탕으로 기술을 선택할 때, 비로소 시스템은 단순한 코드의 집합을 넘어 비즈니스를 지탱하는 견고한 자산이 됩니다.

## Summary

우리가 살펴본 세 가지 통신 패러다임은 각자의 철학에 따라 최적의 전장을 가집니다.

| 구분            | REST                | GraphQL           | gRPC                |
| :-------------- | :------------------ | :---------------- | :------------------ |
| **핵심 철학**   | 자원 지향 (ROA)     | 그래프 탐색       | 프로시저 호출 (RPC) |
| **데이터 형식** | JSON / XML (Text)   | JSON (Text)       | Protobuf (Binary)   |
| **네트워크**    | HTTP 1.1 / 2        | HTTP 1.1 / 2      | HTTP/2 Only         |
| **주요 강점**   | 보편성, 캐싱 효율   | 클라이언트 유연성 | 성능, 타입 안정성   |
| **주요 약점**   | Over/Under-fetching | 서버 구현 복잡도  | 브라우저 지원 미비  |

### 최종 엔지니어링 체크포인트

- **네트워크 비용 vs 개발 속도**: 대역폭 절감과 저지연이 절실하다면 gRPC를, 빠른 생태계 확장이 목표라면 REST를 선택하십시오.
- **클라이언트 제어권**: 다양한 화면 요구사항에 대응해야 하는 프론트엔드 팀의 민첩성이 중요하다면 GraphQL이 훌륭한 대안이 됩니다.
- **시스템 가시성**: 바이너리 기반의 gRPC 도입 시 반드시 모니터링과 로깅 시스템이 해당 포맷을 해석할 수 있는지 먼저 점검해야 합니다.
- **적정 기술의 조화**: 단일 기술에 집착하지 마십시오. BFF 패턴을 통해 내부 트래픽은 gRPC로, 외부 트래픽은 GraphQL/REST로 처리하는 하이브리드 전략이 현대 아키텍처의 정석입니다.

## References

* **[GraphQL]** Thinking in Graphs - https://graphql.org/learn/thinking-in-graphs/
* **[PayPal]** GraphQL a success story for PayPal Checkout - https://developer.paypal.com/community/blog/graphql-a-success-story-for-paypal-checkout/
* **[gRPC]** Core concepts, architecture and lifecycle - https://grpc.io/docs/what-is-grpc/core-concepts/
* **[Protocol Buffers]** Encoding Mechanism - https://protobuf.dev/programming-guides/encoding/
* **[Microsoft]** Backends for Frontends pattern - https://learn.microsoft.com/en-us/azure/architecture/patterns/backends-for-frontends