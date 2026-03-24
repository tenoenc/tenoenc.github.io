---
layout: post
title: "[Retrospect] 문맥 기반 선별적 로깅을 활용한 대용량 트래픽 스토리지 최적화"
date: 2026-03-18 00:34 +0900
categories:
- Retrospect
tags:
- Performance Optimization
- System Design
---

본 문서는 `ONOL` 프로젝트 진행 중 수행된 **"데이터 저장 정책 최적화(Smart Logging)"** 작업에 대한 회고다. 실시간 동영상 스트리밍이나 대용량 파일 전송 시 발생하는 막대한 양의 패킷 데이터로 인한 **스토리지 낭비**와 **Write I/O 병목** 문제를 해결하기 위해, 모든 패킷을 저장하던 기존 방식에서 **'가치 있는 데이터'만 선별하여 저장**하는 방식으로 전환한 과정과 그 성과를 기술한다.

### Problem Statement

초기 프로토타입은 수집된 모든 패킷(All-Capture)을 DB에 저장하도록 설계되었다. 그러나 4K 동영상 시청과 같은 **대용량 트래픽(Elephant Flow)** 환경에서 다음과 같은 문제가 식별되었다.

1. **UDP(QUIC) 트래픽의 스토리지 점유**: 초기 필터링 정책은 TCP 연결 제어에 집중되어 있었다. 이에 따라 YouTube, Netflix 등 최신 스트리밍 서비스가 사용하는 **QUIC(UDP) 프로토콜** 기반의 영상 데이터가 필터링을 우회하여 무제한으로 DB에 저장되는 현상이 발생했다.
2. **낮은 신호 대 잡음비(Low Signal-to-Noise Ratio)**: 저장되는 데이터의 90% 이상이 암호화된 바이너리 페이로드(Payload)로, 복호화 키 없이는 분석 가치가 낮은 암호화 데이터였다.
3. **I/O 병목에 의한 처리 지연**: 초당 수천 건의 무의미한 `INSERT` 쿼리가 발생하며 DB 디스크 I/O가 포화되었고, 이는 전체 시스템의 처리 속도(Throughput) 저하로 이어졌다.

### Solution Strategy

우리는 **"볼륨(Volume)은 메트릭으로, 문맥(Context)은 로그로"**라는 원칙을 수립하고, 기존 TCP 중심의 전략을 보완하여 **프로토콜별 맞춤형 Smart Logging** 전략을 도입했다.

* **기술적 구현**: Redis Atomic Increment를 활용하여 각 세션(Flow)별 누적 패킷 수를 실시간으로 카운팅하고, 이를 기반으로 저장 여부를 결정하는 필터링 로직을 서비스 계층에 배치했다.
* **저장 정책(Policy)**:
    1. **TCP 처리**:
        * **Head**: 통신 초기의 SNI, HTTP Header 등 신원 식별 정보가 포함된 초기 패킷은 저장.
        * **Control**: 연결 상태(`SYN`, `FIN`, `RST`) 추적을 위한 제어 패킷은 저장.
        * **Tail**: 문맥 파악 후 이어지는 단순 데이터 전송은 폐기(Drop).
    2. **UDP 처리**:
        * **Identification**: DNS(Port 53)와 같이 서비스 식별에 필수적인 패킷은 저장.
        * **Streaming Drop**: QUIC 등 대용량 스트리밍 데이터로 식별된 트래픽은 초기 식별 패킷 이후 전량 폐기하여 스토리지 사용량을 제어.

다음은 앞서 설계한 정책이 실제로 적용된 필터링 로직(`PacketLogWorker`)의 핵심 코드다. Redis Atomic Increment를 통해 조회한 `count`를 기반으로, TCP와 UDP 트래픽을 각기 다른 기준으로 선별한다.

```java
private List<PacketLog> filterAndMapPackets(List<PacketEvent> events) {
    // 1. Flow Key 생성 (SrcIP, DstPort 등) 및 Redis Bulk 카운트 조회
    List<String> flowKeys = events.stream()
            .map(tcpSessionTracker::generateFlowKey)
            .toList();
    Map<String, Long> packetCounts = sessionStatePort.incrementPacketCounts(flowKeys);

    List<PacketLog> result = new ArrayList<>();
    int keyIndex = 0;

    for (PacketEvent event : events) {
        String key = flowKeys.get(keyIndex++);
        Long count = packetCounts.getOrDefault(key, 0L);
        boolean shouldSave = false;

        if ("TCP".equalsIgnoreCase(event.protocol())) {
            // [TCP Policy] 제어 패킷(SYN/FIN)은 필수 저장, 데이터는 초반 10개만 저장
            boolean isControl = (event.tcpFlags() & (FLAG_SYN | FLAG_FIN | FLAG_RST)) != 0;
            if (isControl || count <= 10) {
                shouldSave = true;
            }
        } else {
            // [UDP Policy] DNS(53)는 필수 저장, 스트리밍(QUIC) 데이터는 초반 10개 이후 Drop
            if (event.dstPort() == 53 || event.srcPort() == 53) {
                 shouldSave = true; // Context Preservation (DNS)
            } else if (count <= 10) {
                 shouldSave = true; // Volume Control (Drop Tail)
            }
        }

        if (shouldSave) {
            result.add(enrichAndMapToDomain(event));
        }
    }
    return result;
}
```

### Performance Analysis

동일한 트래픽 시나리오(YouTube 4K 스트리밍 및 웹 서핑 혼합)에서 최적화 적용 전후의 시스템 데이터를 비교 분석했다.

#### 1. 데이터 감축 효율

시스템은 트래픽의 성격에 따라 유연하게 저장 비율을 조절하여 스토리지 효율을 개선했다.

* **웹 서핑 (Short-lived Flow)**: 새로운 연결이 빈번한 상황에서는 높은 저장 비율을 유지하며 분석에 필요한 문맥을 온전히 보존했다.
* **스트리밍 (Long-lived Flow)**: 대량의 UDP 패킷이 유입되는 상황에서 `Saved: 0 (100% Reduced)`를 기록했다. 700개 이상의 연속된 영상 데이터 패킷이 유입되었으나 단 한 건도 저장하지 않음으로써 불필요한 스토리지 사용을 차단했다.

#### 2. 처리 속도 향상

Write I/O 부하의 감소는 애플리케이션의 처리 지연 시간(Latency) 단축으로 직결되었다.

| 상황                       | 데이터 처리량 (Batch) | 저장 대상 (Saved) | 처리 시간 (Latency) | 비고                        |
| :------------------------- | :-------------------- | :---------------- | :------------------ | :-------------------------- |
| **저장 위주 (웹 서핑)**    | 593 packets           | 400 (67% 저장)    | **395 ms**          | I/O 발생으로 인한 소요 시간 |
| **필터링 위주 (스트리밍)** | 551 packets           | **0 (0% 저장)**   | **40 ms**           | **약 9.8배 속도 향상**      |

### Conclusion

이번 최적화를 통해 시스템은 단건 수집기에서 문맥 기반의 선별적 로깅 시스템으로 전환되었다.

1. **지속 가능성 확보**: UDP 스트리밍 트래픽까지 포괄하는 필터링을 통해 스토리지 비용을 90% 이상 절감하여, 개인 PC에서도 부담 없이 구동 가능한 경량성을 확보했다.
2. **분석 품질 유지**: 데이터를 대폭 감축했음에도 불구하고, `Who(IP/Domain)`, `When(Time)`, `Status(Flag)` 등 핵심 정보는 100% 유지됨을 검증했다.
3. **아키텍처 유연성**: DB 스키마 변경 없이 애플리케이션 레벨의 정책 변경만으로 시스템의 데이터 수집 거동을 제어할 수 있는 유연한 구조임을 입증했다.