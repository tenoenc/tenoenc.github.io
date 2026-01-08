---
layout: post
title: "[Data Structure] Stack & Queue"
date: 2026-01-08 14:07 +0900
math: true
categories:
- Computer Science
- Data Structure & Algorithm
tags:
- Data Structure
- Stack
- Queue
image:
    path: /assets/img/2026-01-08-16-19-57.png
---

소프트웨어 공학에서 Stack과 Queue는 흔히 '자료구조'라는 추상적 카테고리에 묶여 다루어지곤 합니다. 하지만 시스템 아키텍처의 심부로 내려가면, 이들은 단순한 데이터 보관 방식을 넘어 CPU와 메모리가 상호작용하는 물리적 통로 그 자체가 됩니다. PlanetScale의 분석에 따르면, 현대의 운영체제는 프로세스와 스레드라는 추상화를 통해 하드웨어를 관리하며, 이 과정에서 Stack은 스레드의 '개별적 기억'을, Queue는 시스템 전체의 '질서 있는 흐름'을 담당하는 하드웨어적 인터페이스로 기능합니다.

우리가 코드로 작성하는 한 줄의 함수 호출이나 비동기 메시지 전송은 보이지 않는 곳에서 전압의 변화와 메모리 주소 포인터의 이동으로 치환됩니다. 이 글에서는 자바라는 고수준 언어의 장막을 걷어내고, Stack과 Queue가 실제 물리 세계에서 어떻게 기계의 효율을 결정짓는지, 그리고 그 구조적 선택이 대규모 트래픽 환경에서 어떤 나비효과를 불러오는지 해부해 보겠습니다.

## 🧩 Stack

Stack은 논리적으로 '후입선출(LIFO)'이라는 단순한 규칙을 따르지만, 하드웨어 레벨에서는 **공간 지역성(Spatial Locality)**의 극치라고 불릴 만큼 효율적인 구조를 가집니다. 프로세스가 실행될 때 각 스레드는 고유한 Stack 세그먼트를 할당받으며, 이 공간은 메모리 주소의 높은 곳에서 낮은 곳으로 거꾸로 자라나는 독특한 물리적 특성을 보입니다.

### 하드웨어 스택 포인터와 제어 흐름

함수가 호출될 때마다 CPU는 현재의 실행 지점(Instruction Pointer)을 Stack에 밀어 넣고, 스택 포인터(SP)를 아래로 이동시켜 새로운 '스택 프레임'을 생성합니다. 이 과정은 별도의 복잡한 메모리 할당 알고리즘 없이, 단순히 레지스터 값을 가감하는 것만으로 완료됩니다. 이러한 기계적 단순함 덕분에 Stack은 힙(Heap) 영역의 동적 할당보다 압도적으로 빠른 속도를 보장합니다.

![](/assets/img/2026-01-08-15-44-11.png)

### L1/L2 캐시와 공간 지역성의 정수

CPU의 성능을 결정짓는 핵심 요소는 캐시 적중률(Cache Hit Rate)입니다. Stack은 데이터가 메모리상에 조밀하게 연속적으로 배치되는 특성을 가집니다. CPU가 Stack의 특정 주소에 접근하면, 하드웨어 프리페처(Prefetcher)는 인접한 데이터들 역시 곧 사용될 것임을 예측하여 캐시 라인(통상 64바이트) 전체를 L1 캐시로 끌어올립니다.

함수 내의 지역 변수들이나 매개변수들은 이 좁은 범위 안에 뭉쳐 있으므로, Stack 기반의 연산은 거의 항상 캐시 적중을 보장합니다. 반면 뒤에서 다룰 Node 기반의 Queue나 Heap 영역의 객체들은 메모리 곳곳에 파편화되어 있어, 포인터를 따라갈 때마다 CPU가 데이터를 기다리며 공회전하는 '캐시 미스'를 유발할 가능성이 큽니다.

```text
; 함수 호출 시의 스택 조작 (System V ABI 기준)
_my_function:
    push    rbp             ; 이전 함수의 베이스 포인터를 스택에 보존
    mov     rbp, rsp        ; 현재 스택 포인터를 새로운 베이스 포인터로 설정
    sub     rsp, 32         ; 지역 변수를 위한 32바이트 공간 확보 (스택이 아래로 '자람')

    ; ... 함수 로직 수행 ...
    ; [rbp - 4] 등의 주소로 지역 변수에 접근 (캐시 지역성 높음)

    leave                   ; mov rsp, rbp / pop rbp를 한 번에 수행하여 스택 복구
    ret                     ; 리턴 주소를 pop하여 호출처로 복귀
```

> **메모리 보호와 Stack Guard (Warning)**
> 
> 스택은 물리적으로 제한된 크기를 가집니다. 무한 재귀 호출이 발생하면 스택 포인터는 스택 세그먼트의 경계를 넘어 인접한 데이터 영역을 침범하려 합니다. 운영체제는 이를 방지하기 위해 스택 끝에 'Guard Page'를 두어 즉시 프로세스를 중단시키는데, 이것이 우리가 마주하는 `StackOverflowError`의 물리적 실체입니다.
{: .prompt-warning }

### 컨텍스트 스위칭에서의 무거운 짐

PlanetScale은 컨텍스트 스위칭의 비용 중 상당 부분이 이 Stack 상태를 보존하는 데서 온다고 설명합니다. 스레드가 전환될 때 CPU는 현재 레지스터 값들과 Stack 포인터를 메모리에 백업해야 합니다. 스택의 깊이가 깊을수록, 즉 함수 호출 체인이 복잡할수록 보존해야 할 컨텍스트의 양은 늘어나며, 이는 곧 시스템 전체의 처리량(Throughput) 저하로 이어집니다. 따라서 로우레벨 최적화에서는 Stack의 깊이를 얕게 유지하고 데이터의 복사를 최소화하는 설계가 필수적입니다.

## 🧩 Queue

Stack이 수직적인 기억의 퇴적이라면, Queue는 수평적으로 흐르는 '시간의 정렬'입니다. 선입선출(FIFO)이라는 단순한 규칙 뒤에는 가변적인 입력 속도와 고정된 처리 속도 사이의 간극을 메우려는 엔지니어링의 고뇌가 숨어 있습니다. Amazon Builders' Library의 고찰에 따르면, 시스템 설계에서 Queue는 단순한 데이터 저장소가 아니라 자원의 부하를 분산하고 시스템 붕괴를 막는 '댐'과 같은 역할을 수행합니다.

### 실행 큐(Runqueue)

운영체제 커널의 핵심인 스케줄러는 수많은 프로세스 중 어떤 프로세스에 CPU 자원을 할당할지 결정하기 위해 '실행 큐(Runqueue)'를 관리합니다. 멀티코어 환경에서 각 코어는 자신만의 로컬 큐를 가지며, 이는 코어 간의 자원 경합을 최소화하는 하드웨어적 최적화의 산물입니다.

커널 수준의 큐잉 메커니즘은 단순한 리스트가 아닙니다. 우선순위에 따라 태스크가 재배치되거나, 특정 스레드가 자원을 독점하지 못하도록 공정하게 시간을 배분하는 복잡한 알고리즘이 큐 내부에서 동작합니다. 토스 테크가 TPS 1만 이상의 고부하 환경을 처리할 때 언급한 비동기 이벤트 기반 아키텍처 역시, 본질적으로는 애플리케이션 레벨에서 이러한 커널의 큐잉 메커니즘을 모사하여 데이터베이스의 물리적 한계를 극복하려는 시도입니다.

![](/assets/img/2026-01-08-15-34-27.png)

### 순환 버퍼(Circular Buffer)와 물리적 효율성

큐를 구현할 때 가장 빈번하게 사용되는 아키텍처는 순환 버퍼(Circular Buffer)입니다. 선형적인 배열에서 큐를 구현하면 데이터가 빠져나갈 때마다 남은 요소들을 앞으로 당기는 '메모리 복사' 비용이 발생하지만, 순환 버퍼는 `head`와 `tail` 포인터를 회전시킴으로써 물리적 이동 없이 논리적 순서만을 관리합니다.

하지만 이 영리한 구조에도 하드웨어적 함정이 존재합니다. `head`와 `tail` 포인터가 동일한 캐시 라인(Cache Line)에 위치할 경우, 한쪽이 업데이트될 때마다 다른 쪽 코어의 캐시가 무효화되는 **거짓 공유(False Sharing)** 현상이 발생할 수 있습니다. 로우레벨 시스템 프로그래밍에서는 이를 방지하기 위해 포인터 사이에 의미 없는 데이터(Padding)를 채워 넣어 캐시 성능을 강제로 보존하기도 합니다.

```cpp
// 큐의 크기는 반드시 2의 거듭제곱(2^n)이어야 비트 마스킹 가능
#define BUFFER_SIZE 1024
#define MASK (BUFFER_SIZE - 1)

typedef struct {
    int data[BUFFER_SIZE];
    unsigned int head;
    unsigned int tail;
} CircularQueue;

// 데이터 삽입 (Producer)
void enqueue(CircularQueue* q, int value) {
    // tail을 단순히 증가시키고 마스킹하여 순환 구현
    q->data[q->tail & MASK] = value;
    q->tail++; 
    // 실제 구현에서는 Full 체크 로직이 추가됨
}

// 데이터 추출 (Consumer)
int dequeue(CircularQueue* q) {
    int value = q->data[q->head & MASK];
    q->head++;
    return value;
}
```

### I/O 가속을 위한 하드웨어 큐

현대 하드웨어 아키텍처에서 Queue는 소프트웨어의 전유물이 아닙니다. 네트워크 카드(NIC)나 SSD 컨트롤러 내부에는 하드웨어적으로 구현된 수많은 큐가 존재합니다. 

네트워크 패킷이 초당 수백만 개씩 쏟아질 때, CPU가 이를 실시간으로 처리하는 것은 불가능합니다. 이때 NIC 내부의 수신 큐(RX Queue)는 패킷을 임시로 적재하여 CPU에 인터럽트를 보내기 전까지의 완충 지대 역할을 합니다. 만약 이 큐의 크기가 너무 작으면 패킷 유실(Drop)이 발생하고, 반대로 너무 크면 데이터가 큐에 머무는 시간이 길어져 **버퍼블로트(Bufferbloat)** 현상, 즉 처리량은 높지만 지연 시간(Latency)이 기하급수적으로 늘어나는 역설적인 상황에 직면하게 됩니다.

> **큐의 크기와 지연 시간의 트레이드오프 (Info)**
> 
> 큐는 시스템의 가용성을 높여주지만, 공짜는 아닙니다. 큐가 길어질수록 요청이 처리되기까지 대기하는 '큐잉 지연(Queuing Delay)'이 발생합니다. Amazon Builders' Library에서는 이를 제어하기 위해 큐의 크기를 제한(Bounded Queue)하고, 넘치는 요청에 대해서는 빠르게 실패(Fail-fast) 처리하는 것이 시스템 전체의 안정성을 위해 더 나은 선택이라고 조언합니다.
{: .prompt-info }

### 캐시 미스와 지역성 파괴의 위험

Stack이 가진 완벽한 공간 지역성과 달리, Queue는 생산자(Producer)와 소비자(Consumer)가 데이터의 양 끝단을 건드립니다. 만약 큐의 크기가 CPU 캐시 용량보다 커지면, 생산자가 방금 써넣은 데이터는 소비자가 읽으러 오기 전에 이미 캐시에서 쫓겨날 가능성이 높습니다. 이는 곧 메모리 버스(Memory Bus)에 직접 접근해야 하는 비용을 발생시키며, 고성능 시스템에서 큐의 크기를 무조건 크게 잡을 수 없는 물리적 이유가 됩니다.

## Java를 통한 물리적 추상화의 구현

자바에서 Stack과 Queue를 다루는 방식은 언뜻 단순해 보이지만, JVM(Java Virtual Machine)의 메모리 관리 모델 위에서 이들이 어떻게 물리적으로 배치되는지 이해하는 것은 성능 최적화의 관건입니다. Oracle Java Documentation 명세에 따르면, 자바는 개발자에게 포인터 직접 조작을 허용하지 않는 대신 고도로 추상화된 컬렉션 프레임워크를 제공합니다. 그러나 이 추상화 뒤에는 메모리 파편화와 참조 추적이라는 명확한 하드웨어적 비용이 존재합니다.

### Array 기반 구현

자바의 `ArrayDeque`나 `ArrayList`를 기반으로 구현된 Stack과 Queue는 물리적으로 연속된 메모리 공간을 점유합니다. 이는 앞서 살펴본 CPU 캐시 효율성을 극대화하는 방식입니다. 자바의 배열은 객체이므로 힙(Heap) 영역에 생성되지만, 배열 내부의 각 요소는 메모리상에 조밀하게 붙어 있습니다. 

배열 기반 구조의 핵심은 **인덱스 산술 연산**입니다. 특정 요소의 위치를 찾기 위해 메모리를 헤맬 필요 없이, '시작 주소 + (인덱스 * 데이터 크기)'라는 단순 계산만으로 데이터에 즉시 접근합니다. 하지만 동적으로 크기가 확장될 때(Resizing), 기존 배열의 모든 데이터를 새로운 더 큰 배열로 복사해야 하는 물리적 비용이 발생합니다. 이는 순간적인 Latency 스파이크를 유발하며, 토스 테크가 강조한 TPS 1만 이상의 환경에서는 초기 용량(Initial Capacity)을 적절히 설정하여 이 복사 과정을 최소화하는 것이 필수적입니다.

```java
public class OptimizedArrayQueue<E> {
    private E[] elements;
    private int head;
    private int tail;
    private int size;

    @SuppressWarnings("unchecked")
    public OptimizedArrayQueue(int capacity) {
        // 초기 용량을 설정하여 불필요한 Resizing 방지 (Toss Tech의 TPS 대응 전략)
        this.elements = (E[]) new Object[capacity];
    }

    public void enqueue(E e) {
        if (size == elements.length) {
            doubleCapacity(); // 자원 고갈 시 확장
        }
        elements[tail] = e;
        tail = (tail + 1) % elements.length;
        size++;
    }

    private void doubleCapacity() {
        int n = elements.length;
        int p = head;
        int r = n - p; 
        int newCapacity = n << 1; // 2배 확장
        Object[] a = new Object[newCapacity];
        
        // System.arraycopy는 하드웨어 가속을 받는 고속 복사 연산임
        System.arraycopy(elements, p, a, 0, r);     // head부터 끝까지 복사
        System.arraycopy(elements, 0, a, r, p);     // 처음부터 head 전까지 복사
        
        this.elements = (E[]) a;
        this.head = 0;
        this.tail = n;
    }
}
```

### Node 기반 구현과 포인터의 저주

반면 `LinkedList`로 구현된 Queue나 Stack은 메모리의 '파편화'를 전제로 합니다. 각 데이터는 `Node`라는 객체에 감싸져 힙 영역 곳곳에 흩어집니다. 이 방식의 가장 큰 문제는 **포인터 체이닝(Pointer Chasing)**입니다. 다음 데이터를 읽기 위해 현재 노드가 가진 참조 주소를 따라가야 하는데, 이 주소가 가리키는 곳이 CPU 캐시에 없을 확률이 매우 높습니다. 

또한, 자바 객체는 최소 12~16바이트의 객체 헤더(Object Header)를 가집니다. 단순한 `int` 값 하나를 큐에 넣더라도 노드 객체 자체의 오버헤드와 전후방 참조 포인터가 결합되어 실제 데이터보다 수 배 많은 메모리를 낭비하게 됩니다. 이는 가비지 컬렉터(GC)가 추적해야 할 객체 수를 늘려, 시스템 전체의 'STW(Stop-The-World)' 시간을 길게 만드는 주범이 됩니다.

![](/assets/img/2026-01-08-15-34-38.png)

### synchronized와 CAS

자바의 레거시 클래스인 `java.util.Stack`은 모든 메서드에 `synchronized` 키워드가 붙어 있습니다. 이는 하드웨어 레벨에서 특정 스레드가 락(Lock)을 획득할 때까지 다른 스레드들을 대기 상태(Blocked)로 만드는데, 컨텍스트 스위칭 비용을 발생시키는 매우 무거운 작업입니다.

현대적인 자바 아키텍처에서는 이를 대신해 `java.util.concurrent` 패키지의 **CAS(Compare-And-Swap)** 기반 자료구조를 사용합니다. `ConcurrentLinkedQueue`와 같은 구현체는 락을 거는 대신, 하드웨어의 원자적 연산을 이용하여 '내가 읽었을 때의 값과 지금 쓰려는 시점의 값이 같을 때만' 업데이트를 수행합니다. 락 없이도 데이터 무결성을 보장하는 이 방식은 고부하 환경에서 스레드 간의 경합을 드라마틱하게 줄여줍니다.

> **자바에서 Stack 대신 Deque를 사용해야 하는 이유**
> 
> 자바 공식 문서에서는 `java.util.Stack` 대신 `java.util.Deque` 인터페이스의 구현체인 `ArrayDeque`를 사용할 것을 권장합니다. `Stack`은 `Vector`를 상속받아 불필요한 동기화 비용이 발생하며, 상속 구조상 리스트의 중간에 데이터를 삽입하는 등 Stack의 본질을 해치는 동작이 가능하기 때문입니다. 객체 지향의 설계 원칙과 물리적 성능 두 마리 토끼를 잡기 위해서는 `Deque`가 정답입니다.
{: .prompt-tip }

## 자원의 고갈과 지연의 상관관계

엔지니어링의 본질은 무조건적인 최적화가 아닌, 주어진 제약 조건 내에서의 영리한 선택(Trade-off)에 있습니다. Stack과 Queue는 각각 시스템의 안정성과 성능을 담보하는 강력한 도구이지만, 그 물리적 한계를 오해했을 때 시스템은 가장 치명적인 방식으로 무너집니다.

### Stack Overflow

Stack은 '속도'를 위해 '유연성'을 희생한 구조입니다. 앞서 살펴본 것처럼 Stack은 정해진 크기의 메모리 세그먼트 내에서만 동작합니다. 함수가 자기 자신을 끝없이 호출하는 재귀(Recursion)나, 과도하게 큰 지역 변수를 선언하는 행위는 하드웨어가 허용한 경계를 순식간에 갉아먹습니다.

이 현상은 단순한 소프트웨어 오류가 아니라, 물리적 메모리 압착에 가깝습니다. PlanetScale의 분석에 따르면, 현대 OS는 페이지 테이블을 통해 각 스레드의 스택을 관리하는데, 스택이 가득 차서 인접한 메모리 페이지를 침범하려 할 때 발생하는 하드웨어 예외가 바로 Stack Overflow의 실체입니다. 이는 시스템이 제어 흐름을 상실하지 않도록 강제로 멈추는 일종의 '퓨즈' 역할을 수행합니다.

### Bufferbloat

Queue의 설계에서 가장 흔히 저지르는 실수는 "버퍼가 크면 클수록 안전할 것"이라는 믿음입니다. 그러나 네트워크와 분산 시스템 아키텍처에서는 이를 **버퍼블로트(Bufferbloat)**라는 치명적인 독으로 간주합니다. 큐가 너무 크면, 시스템에 과부하가 걸렸을 때 패킷이나 요청들이 큐에 아주 오랫동안 머물게 됩니다.

이때 발생하는 큐잉 지연(Queuing Delay)은 사용자 입장에서 서비스가 '죽은 것'과 다름없는 상태를 만듭니다. Amazon Builders' Library의 통찰에 의하면, 요청이 처리되지 못하고 큐에 쌓여 있는 동안 호출자는 타임아웃을 겪고 다시 재시도(Retry)를 보냅니다. 이는 이미 가득 찬 큐에 더 많은 부하를 가중시키는 '재시도 폭풍(Retry Storm)'을 유발하며, 시스템 전체의 가용성을 0으로 수렴하게 만듭니다.

![](/assets/img/2026-01-08-15-34-47.png)

> **안정적인 시스템을 위한 Fail-fast 전략**
> 
> 토스 테크가 TPS 1만 이상의 환경에서 캐시와 큐를 적용하며 얻은 교훈은, 시스템이 모든 요청을 감당하려 애쓰는 대신 '버틸 수 없는 요청'은 빠르게 거절(Fail-fast)하는 것이 전체의 생존을 위해 낫다는 것입니다. 큐의 크기를 유한하게 설정하고(Bounded Queue), 한계를 넘어서는 순간 에러를 반환하는 설계가 아키텍처적 완성도를 결정합니다.
{: .prompt-warning }

## Summary

Stack과 Queue는 단순히 데이터를 쌓고 줄을 세우는 논리적 도구가 아닙니다. Stack은 CPU 캐시의 공간 지역성을 극대화하여 연산의 마찰을 줄이는 **하드웨어 가속기**이며, Queue는 비동기 환경에서 입력의 불확실성을 완충하고 질서를 부여하는 **시스템의 댐**입니다.

성공적인 아키텍처는 Stack의 깊이를 제어하여 컨텍스트 스위칭 비용을 아끼고, Queue의 크기를 전략적으로 제한하여 버퍼블로트의 늪에 빠지지 않는 균형 감각에서 탄생합니다. 우리가 작성하는 코드가 물리적인 메모리 주소와 CPU 사이클 위에서 어떻게 춤추고 있는지 이해할 때, 비로소 진정으로 견고한 소프트웨어를 설계할 수 있습니다.

## References

* [[PlanetScale] Processes and Threads](https://planetscale.com/blog/processes-and-threads)
* [[Toss Tech] 캐시를 적용하기 까지의 험난한 길 (TPS 1만 안정적으로 서비스하기)](https://toss.tech/article/34481)
* [[Amazon Builders' Library] Timeouts, retries and backoff with jitter](https://aws.amazon.com/ko/builders-library/timeouts-retries-and-backoff-with-jitter/)
* [[Oracle Docs] Java Queue Interface](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Queue.html)
* [[Oracle Docs] Java Stack Class](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Stack.html)