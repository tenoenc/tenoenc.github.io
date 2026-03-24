---
layout: post
title: "[Retrospect] Redis N+1 문제 해결을 통한 실시간 대용량 패킷 분석 파이프라인 구축"
date: 2026-03-17 21:37 +0900
categories:
- Retrospect
tags:
- Performance Optimization
- Redis
- N+1 Problem
---

본 회고는 로컬 네트워크 패킷 분석 시스템 개발 중 발생한 데이터 처리 지연 현상을 분석하고, 이를 아키텍처 관점에서 해결한 과정을 기술한다. 특히 대용량 트래픽 환경에서 발생한 **Redis N+1 입출력 문제**를 진단하고, **Pipelining 및 Bulk 연산**을 도입하여 시스템 처리량을 약 700% 향상시킨 경험을 중점적으로 다룬다.

### Problem Context

패킷 수집 및 분석 파이프라인의 부하 테스트를 위해 라이브 동영상 스트리밍(TCP 고속 트래픽) 환경에서 시스템을 가동했다. 초기 10분간은 정상 작동했으나, 시간이 지날수록 **데이터 수집 시점과 DB 저장 시점 간의 괴리(Lag)**가 기하급수적으로 증가했다.

* **증상**: 시스템 가동 20분 경과 시점에서 약 8분의 데이터 처리 지연 발생.
* **지표**: 배치 사이즈 500개 처리 시 평균 소요 시간이 약 1,250ms로 측정됨.
* **영향**: 실시간 위협 탐지 및 세션 추적 기능이 불능 상태에 빠짐(Backpressure 발생).

### Root Cause Analysis

병목 구간을 특정하기 위해 각 구성 요소(Kafka, Application, DB, Redis)를 분석한 결과, **Application과 Redis 간의 비효율적인 통신 패턴**이 주원인임을 확인했다.

* **구조적 결함**: `PacketService`가 Kafka로부터 수신한 패킷 리스트를 순회하며, 개별 패킷마다 Redis에 동기 요청을 보내고 있었다.
* **N+1 문제**: 패킷 1개를 분석하기 위해 `위협 탐지(3회)` + `세션 조회(1회)` + `세션 갱신(1회)` 등 평균 5회의 Redis 네트워크 호출이 발생했다.
* 배치 사이즈가 3,000개일 경우, 단일 트랜잭션 내에서 **15,000회의 네트워크 RTT**가 발생.
* 로컬 환경임에도 불구하고, I/O Wait 시간이 CPU 연산 시간을 압도하며 전체 처리 속도를 저하시킴.



### Solution Strategy

네트워크 I/O 비용을 최소화하기 위해 단건 처리 방식을 **일괄 처리** 방식으로 리팩토링했다.

#### 1. Redis Pipelining 도입

기존에는 루프 내에서 Redis 명령을 순차적으로 호출하여 Network I/O가 패킷 수(N)에 비례해 증가했다. 이를 `executePipelined`로 감싸, 수천 개의 명령어를 단 한 번의 패킷으로 전송하도록 변경했다.

```java
// [Before] N+1 문제 발생 코드 (Legacy)
// 패킷 1개당 Redis와 3번 통신 (RTT * 3 * N)
for (PacketEvent packet : packets) {
    redisTemplate.opsForSet().add(key, port); // 1. 접속 기록
    redisTemplate.expire(key, WINDOW);        // 2. TTL 갱신
    Long count = redisTemplate.opsForSet().size(key); // 3. 개수 확인
    if (count == threshold) { ... }
}

// [After] Pipelining 적용 코드 (Optimized)
// 수천 개의 명령을 모아서 1번만 통신 (RTT * 1)
List<Object> results = redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
    for (PacketEvent packet : candidates) {
        byte[] key = serializer.serialize(packet.srcIp());
        byte[] port = serializer.serialize(String.valueOf(packet.dstPort()));
        
        connection.sAdd(key, port); // 명령어를 큐에 적재 (전송 X)
        connection.expire(key, WINDOW.getSeconds());
        connection.sCard(key);
    }
    return null; // 이 시점에 일괄 전송 (Flush)
});
```

#### 2. Session State Bulk 연산 구현

Spring Data Redis의 `MultiGet`과 `Pipeline SET`을 활용하여 세션 상태 관리 로직을 최적화했다. 특히 `RedisStringCommands`를 직접 사용하여 불필요한 직렬화 오버헤드를 줄였다.

```java
// Bulk Update 구현체 (Optimized)
@Override
public void updateSessionStates(Map<String, String> updates) {
    redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
        for (Map.Entry<String, String> entry : updates.entrySet()) {
            // Deprecated된 setEx 대신 Native Command 사용
            connection.stringCommands().set(
                    serializer.serialize(KEY_PREFIX + entry.getKey()),
                    serializer.serialize(entry.getValue()),
                    Expiration.seconds(SESSION_TTL.getSeconds()),
                    RedisStringCommands.SetOption.upsert()
            );
        }
        return null;
    });
}
```

#### 3. 서비스 계층의 로직 변경

서비스 계층에서는 DB 접근 로직을 루프 밖으로 끄집어내어, **"Bulk Read → In-Memory Calculation → Bulk Write"** 패턴으로 재구성했다.

```java
// In-Memory 상태 전이 로직 (Optimized)
// 1. Bulk Read (네트워크 1회)
Map<String, String> currentStates = sessionStatePort.getSessionStates(flowKeys);

// 2. Memory Calculation (네트워크 0회)
for (int i = 0; i < tcpEvents.size(); i++) {
    String currentState = currentStates.get(key);
    String nextState = tracker.determineNextState(currentState, event);
    if (!nextState.equals(currentState)) {
        updates.put(key, nextState); // 변경분만 수집
    }
}

// 3. Bulk Write (네트워크 1회)
sessionStatePort.updateSessionStates(updates);
```

### Performance Metrics

동일한 하드웨어 및 트래픽 환경에서 최적화 전후의 성능 지표를 비교 측정했다. (측정 도구: Application Log StopWatch & Redis Monitor)

| 측정 지표 (Metrics)            | 최적화 전 (Legacy) | 최적화 후 (Optimized) | 증감률            |
| ------------------------------ | ------------------ | --------------------- | ----------------- |
| **평균 처리 시간 (Batch 500)** | ~1,250 ms          | **~220 ms**           | **약 83% 단축**   |
| **초당 처리량 (TPS)**          | ~390 TPS           | **~2,600 TPS**        | **약 6.7배 향상** |
| **시스템 지연 (Lag)**          | 지속적 증가 (8분+) | **0초 (실시간 유지)** | **완전 해소**     |

### Conclusion

이번 최적화 과정을 통해 대용량 데이터 처리 시스템에서 **I/O 비용 관리**가 성능에 미치는 절대적인 영향을 재확인했다.

1. **네트워크 비용의 간과**: 로컬 개발 환경이라 할지라도 수만 번의 반복적인 네트워크 호출은 성능 저하가 발생한다. 분산 시스템 설계 시 RTT를 줄이는 것이 최우선 과제임을 명심해야 한다.
2. **배치 처리의 중요성**: 스트림 데이터 처리라 하더라도, 내부 로직은 가능한 한 마이크로 배치 단위로 그룹화하여 처리하는 것이 효율적이다.
3. **TDD의 가치**: 구조를 대대적으로 변경하는 리팩토링 과정에서, 사전에 작성된 단위 테스트가 변경에 따른 사이드 이펙트를 감지하고 로직의 정합성을 유지하는 역할을 수행했다.

본 프로젝트는 단순한 기능 구현을 넘어, 고성능 처리를 위한 아키텍처 레벨의 고민과 검증 과정을 통해 대용량 데이터 처리 시 I/O 최적화 설계의 중요성을 확인했다.