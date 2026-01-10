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

## Stack

기억은 수직으로 쌓입니다. 우리가 웹 브라우저의 뒤로가기 버튼을 누르거나, 복잡한 수식을 계산하거나, 혹은 함수를 호출하고 다시 돌아올 지점을 찾는 모든 행위의 이면에는 '가장 나중에 들어온 것이 가장 먼저 나간다'는 LIFO(Last-In, First-Out)의 단순하고도 강력한 질서가 자리 잡고 있습니다. 

스택은 추상적인 개념이지만, 이를 물리적 메모리 위에 구현하는 순간 우리는 '공간의 연속성'과 '확장의 유연성'이라는 두 갈래 길 앞에 서게 됩니다.

### LIFO의 논리와 제약

스택의 본질은 '제약'에 있습니다. 데이터의 삽입(Push)과 삭제(Pop)를 단 하나의 통로인 'Top'으로 제한함으로써, 자료구조는 비로소 예측 가능한 상태 추적 도구가 됩니다. 이는 마치 책상 위에 책을 한 권씩 쌓아 올리는 것과 같습니다. 가장 아래에 있는 책을 꺼내기 위해서는 그 위에 쌓인 모든 책을 먼저 치워야 합니다. 

이러한 수직적 구조는 '현재'를 처리하기 위해 '과거'를 안전하게 발밑에 보관해야 하는 알고리즘에 최적화되어 있습니다. 재귀 함수가 호출될 때마다 이전 함수의 상태를 스택 프레임(Stack Frame)에 쌓아두는 이유도 바로 여기에 있습니다.

### 배열 기반 Stack

가장 직관적인 구현은 고정된 크기의 배열을 사용하는 것입니다. 메모리 상에서 데이터가 빈틈없이 나란히 배치되기 때문에, CPU는 다음 데이터가 어디에 있을지 명확히 예측할 수 있습니다.

![](/assets/img/2026-01-10-19-43-04.png)

배열 기반 스택은 인덱스 접근만으로 모든 연산이 $O(1)$에 완료되는 압도적인 성능을 보여줍니다. 하지만 치명적인 한계가 있습니다. 바로 '유리 천장'이라 불리는 Stack Overflow입니다. 선언 시점에 정해진 메모리 경계를 넘어서는 순간, 시스템은 더 이상 데이터를 받아들일 수 없게 됩니다. 이는 하드웨어 자원이 극도로 제한된 임베디드 환경에서 여전히 중요한 설계 고려 요소입니다.

### 동적 배열과 분할 상환 분석

정적인 한계를 극복하기 위해 현대 프로그래밍 언어의 표준 라이브러리(예: Java의 ArrayList, Python의 List)는 동적 배열(Dynamic Array) 전략을 취합니다. 공간이 가득 차면, 기존 배열보다 2배 큰 새로운 공간을 할당하고 모든 데이터를 복사하는 방식입니다.

여기서 의문이 생깁니다. "데이터를 복사하는 $O(n)$의 비용이 발생하는데, 과연 스택의 효율성을 유지할 수 있는가?"

Interview Cake의 분석에 따르면, 이는 '분할 상환(Amortized)'의 관점에서 정당화됩니다. 배열의 크기를 2배로 늘릴 때 발생하는 큰 비용은, 그 직후에 이어질 수많은 $O(1)$ 연산들이 나누어 가집니다. $n$번의 Push 중 단 한 번만 발생하는 $O(n)$의 복사 비용을 전체 연산 횟수로 나누면, 개별 연산의 평균 비용은 여전히 상수 시간 $O(1)$에 수렴하게 됩니다. 우리는 가끔 발생하는 '이사 비용'을 감수함으로써 무한히 확장 가능한 스택의 편리함을 얻는 셈입니다.

### 연결 리스트 기반 Stack

배열의 연속적인 메모리 할당이 부담스럽다면, 연결 리스트(Linked List)가 대안이 됩니다. 각 데이터 원소(Node)는 자신의 값과 함께 다음 원소의 주소를 가리키는 포인터를 가집니다.

연결 리스트 기반 스택은 이론적으로 메모리가 허용하는 한 무한히 커질 수 있으며, 배열처럼 한꺼번에 큰 공간을 미리 점유할 필요도 없습니다. 하지만 세상에 공짜는 없습니다. 포인터를 저장하기 위한 추가 메모리가 필요하며, 무엇보다 '캐시 지역성(Cache Locality)'을 잃게 됩니다. 데이터가 메모리 여기저기 흩어져 있기 때문에 CPU는 다음 노드를 찾기 위해 매번 메인 메모리까지 손을 뻗어야 하며, 이는 배열 기반 구현에 비해 유의미한 성능 저하를 야기합니다.

```java
/**
 * 고정 크기 배열을 기반으로 한 고성능 제네릭 스택입니다.
 * LIFO(Last-In, First-Out) 원칙을 물리적 인덱스 제어로 구현합니다.
 */
public class ArrayStack<T> {
    private T[] storage;
    private int top;
    private final int capacity;

    @SuppressWarnings("unchecked")
    public ArrayStack(int capacity) {
        this.capacity = capacity;
        // Java 제네릭 배열 생성 제약으로 인해 Object 배열 생성 후 캐스팅
        this.storage = (T[]) new Object[capacity];
        this.top = -1; // 빈 스택을 의미하는 초기 인덱스
    }

    public void push(T item) {
        if (isFull()) {
            throw new RuntimeException("Stack Overflow: 스택의 가용 용량을 초과했습니다.");
        }
        storage[++top] = item;
    }

    public T pop() {
        if (isEmpty()) {
            throw new RuntimeException("Stack Underflow: 빈 스택에서 데이터를 꺼낼 수 없습니다.");
        }
        T item = storage[top];
        storage[top--] = null; // 메모리 누수 방지를 위한 참조 해제
        return item;
    }

    public T peek() {
        return isEmpty() ? null : storage[top];
    }

    public boolean isEmpty() {
        return top == -1;
    }

    public boolean isFull() {
        return top == capacity - 1;
    }

    public int size() {
        return top + 1;
    }
}
```

> **속도의 연속성과 공간의 유연성**
>
> 데이터의 최대 크기를 예측할 수 있고 극강의 성능이 필요하다면 **배열 기반**을, 데이터의 증감을 예측하기 어렵고 메모리를 동적으로 관리해야 한다면 **연결 리스트 기반**을 선택하십시오. 대부분의 범용적인 상황에서는 동적 배열 기반의 스택이 가장 합리적인 트레이드오프를 제공합니다.
{: .prompt-tip }

## Queue

스택이 '되돌아가기 위한 기억'이라면, 큐는 '앞으로 나아가기 위한 질서'입니다. 먼저 온 데이터가 먼저 처리되는 FIFO(First-In, First-Out)의 원칙은 컴퓨터 시스템에서 공정성(Fairness)과 흐름 제어(Flow Control)를 담당하는 핵심 기저입니다. 

네트워크 패킷의 대기열부터 운영체제의 프로세스 스케줄링에 이르기까지, 큐는 폭주하는 요청을 질서 정연하게 줄 세워 시스템의 붕괴를 막는 방파제 역할을 수행합니다.

### FIFO와 선형 질서

큐의 논리는 지극히 상식적입니다. 맛집 앞에 늘어선 줄처럼, 먼저 도착한 사람이 먼저 서비스를 받는 구조입니다. 이는 데이터가 발생한 '시간적 순서'를 보존해야 할 때 사용됩니다. 

하지만 이 상식적인 논리를 물리적 메모리(배열)에 그대로 투영하면 예기치 못한 비효율의 늪에 빠지게 됩니다. 데이터를 하나 꺼낼 때마다(Dequeue), 뒤에 서 있는 모든 데이터를 한 칸씩 앞으로 당겨야(Shift) 하기 때문입니다. 원소의 개수가 $n$개라면 매번 $O(n)$의 시간이 소요되며, 이는 대규모 시스템에서 치명적인 병목 현상을 야기합니다.

### Circular Queue

엔지니어들은 이 $O(n)$의 마찰력을 제거하기 위해 배열의 끝과 처음을 이어 붙이는 기하학적 해결책을 고안했습니다. 바로 환형 큐(Circular Queue)입니다.

![](/assets/img/2026-01-10-19-41-53.png)

환형 큐에서는 데이터를 앞으로 당기지 않습니다. 대신 '출구(Front)'와 '입구(Rear)'를 가리키는 포인터만 이동시킵니다. 배열의 마지막 인덱스에 도달하면 다음 인덱스를 다시 0으로 돌려보내는 모듈러(`%`) 연산이 이 마법의 핵심입니다.

$rear = (rear + 1) \pmod{size}$

이 수식을 통해 큐는 고정된 크기의 메모리 안에서 끊임없이 순환하며 삽입과 삭제를 모두 $O(1)$에 처리해냅니다. 다만, '가득 찬 상태(Full)'와 '비어 있는 상태(Empty)'를 구분하기 위해 배열의 한 칸을 비워두거나 별도의 플래그를 관리해야 하는 정교한 설계가 필요합니다.

### 연결 리스트 기반 Queue

배열의 크기 제한에서 자유롭고 싶다면 연결 리스트를 사용합니다. 스택과 달리 큐는 양쪽 끝에서 작업이 일어나므로, 리스트의 '머리(Head)'와 '꼬리(Tail)'를 모두 가리키는 두 개의 포인터를 유지하는 것이 핵심입니다.

이 방식은 환형 큐와 같은 복잡한 인덱스 계산이나 공간 낭비가 없으며, 이론적으로 무한한 확장이 가능합니다. 하지만 각 노드가 다음 노드의 주소를 담아야 하므로 배열 대비 약 2배 이상의 메모리를 소모하며, 빈번한 노드 생성과 삭제는 가비지 컬렉션(GC)이나 메모리 할당자에게 부담을 줍니다.

```java
/**
 * 고정된 크기의 배열 내에서 인덱스를 순환시켜 메모리 효율을 극대화한 순환 큐입니다.
 * FIFO(First-In, First-Out) 논리를 기반으로 동작합니다.
 */
public class CircularQueue<T> {
    private final T[] storage;
    private int front; // 데이터를 꺼낼 위치
    private int rear;  // 데이터를 넣을 위치
    private int count; // 현재 큐에 담긴 요소의 개수
    private final int capacity;

    @SuppressWarnings("unchecked")
    public CircularQueue(int capacity) {
        this.capacity = capacity;
        this.storage = (T[]) new Object[capacity];
        this.front = 0;
        this.rear = 0;
        this.count = 0;
    }

    public void enqueue(T item) {
        if (isFull()) {
            throw new RuntimeException("Queue Overflow: 자원이 고갈되었습니다.");
        }
        storage[rear] = item;
        // 배열의 끝에 도달하면 다시 0번 인덱스로 회전
        rear = (rear + 1) % capacity;
        count++;
    }

    public T dequeue() {
        if (isEmpty()) {
            throw new RuntimeException("Queue Underflow: 처리할 요청이 존재하지 않습니다.");
        }
        T item = storage[front];
        storage[front] = null; // GC를 돕기 위한 참조 해제
        // 인덱스를 순환시켜 논리적 연속성 확보
        front = (front + 1) % capacity;
        count--;
        return item;
    }

    public boolean isEmpty() {
        return count == 0;
    }

    public boolean isFull() {
        return count == capacity;
    }

    public int size() {
        return count;
    }
}
```

### 하드웨어와 자료구조의 마찰

큐를 구현할 때 가장 간과하기 쉬운 지점은 '데이터의 이동성'입니다. CPU 캐시는 연속적인 메모리 접근에 최적화되어 있습니다. 환형 큐(배열 기반)는 포인터가 한 방향으로 순차적으로 움직이므로 캐시 적중률(Cache Hit Rate)이 매우 높습니다. 

반면 연결 리스트 기반 큐는 노드들이 메모리 도처에 파편화되어 존재할 가능성이 큽니다. 논리적으로는 똑같은 $O(1)$일지라도, 실제 하드웨어 위에서 구동되는 속도는 수 배에서 수십 배까지 차이 날 수 있습니다. 현대의 고성능 시스템들이 가급적 고정 크기의 환형 큐(Ring Buffer)를 선호하는 이유는 바로 이 '물리적 속도' 때문입니다.

> **처리량 최적화와 캐시 미스의 상관관계**
> 
> 단순히 "데이터를 줄 세운다"는 목적을 넘어, 시스템의 처리 속도와 메모리 대역폭을 고려하세요.
> 
> 큐의 크기가 예측 가능하다면 **고정 크기 환형 큐**가 정답이며, 예측 불가능한 스파이크 트래픽이 잦다면 **연결 리스트 기반** 혹은 **동적 확장 큐**를 고려해야 합니다.
{: .prompt-info }

## 상호 치환 구현

자료구조의 세계에서 스택과 큐는 서로 대척점에 서 있는 것처럼 보입니다. 하나는 과거를 쌓아두고, 다른 하나는 미래를 향해 흐릅니다. 하지만 논리적 기교를 발휘하면 이 둘은 서로를 완벽하게 흉내 낼 수 있습니다. 이 '상호 치환' 과정은 단순히 알고리즘 퍼즐을 푸는 재미를 넘어, 우리가 사용하는 시스템의 추상화가 얼마나 유연하며 그 이면에 어떤 비용이 숨어있는지를 극명하게 보여줍니다.

### 두 개의 스택이 만드는 시간의 질서

스택으로 큐를 만드는 과정은 '기억을 뒤집는 연금술'과 같습니다. 스택은 데이터를 넣은 순서의 역순으로 뱉어내지만, 그 결과를 다른 스택에 다시 쏟아부으면 원래의 순서가 복원됩니다. 

이 구현의 핵심은 역할을 분담한 두 개의 스택, `inStack`과 `outStack`입니다.

1. **Enqueue**: 데이터를 무조건 `inStack`에 쌓습니다. 이 단계에서는 스택의 본질인 LIFO를 그대로 따릅니다.
2. **Dequeue**: `outStack`에서 데이터를 꺼냅니다. 만약 `outStack`이 비어 있다면, `inStack`에 쌓인 모든 데이터를 하나씩 팝(Pop)하여 `outStack`으로 옮깁니다. 이 과정에서 데이터의 물리적 순서가 완전히 뒤집히며 FIFO의 질서가 탄생합니다.

![](/assets/img/2026-01-10-19-42-05.png)

Stanford CS166의 분석에 따르면, 이 구조는 얼핏 보기에 데이터를 옮길 때마다 $O(n)$의 비용이 발생하여 비효율적으로 보일 수 있습니다. 하지만 분할 상환 분석(Amortized Analysis)을 적용하면 놀라운 결과가 도출됩니다. 모든 데이터 원소는 일생 동안 단 한 번 `inStack`에 들어가고, 단 한 번 `outStack`으로 옮겨지며, 단 한 번 `outStack`에서 나옵니다. 즉, $n$개의 요청에 대해 전체 연산량은 $3n$에 불과하며, 개별 연산의 평균 비용은 여전히 $O(1)$입니다.

### 두 개의 큐가 흉내 내는 수직적 적층

반대로 두 개의 큐를 사용하여 스택을 구현하는 것은 훨씬 고통스럽고 비싼 대가를 치러야 합니다. 큐는 데이터의 순서를 보존하려는 관성이 강하기 때문에, 가장 나중에 들어온 데이터를 가장 먼저 꺼내기 위해서는 나머지 데이터를 모두 뒤로 돌려보내야 합니다.

새로운 데이터가 들어올 때마다(Push), 보조 큐를 활용하여 기존 데이터를 모두 뒤로 밀어내고 새 데이터를 가장 앞(Front)에 배치하는 전략을 취합니다. 이 경우 매번 삽입할 때마다 $O(n)$의 비용이 발생합니다. 혹은 꺼낼 때(Pop) 마지막 원소만 남을 때까지 다른 큐로 데이터를 옮기는 방식을 택할 수도 있지만, 어떤 방식을 선택하든 스택으로 큐를 만들 때와 같은 '분할 상환의 마법'은 일어나지 않습니다.

```java
/**
 * 노드 간의 양방향 연결을 통해 크기 제약 없이 확장 가능한 데크입니다.
 * 스택(LIFO)과 큐(FIFO)의 역할을 동시에 수행할 수 있는 다목적 자료구조입니다.
 */
public class LinkedDeque<T> {
    private static class Node<T> {
        private T data;
        private Node<T> prev;
        private Node<T> next;

        Node(T data) {
            this.data = data;
        }
    }

    private Node<T> head;
    private Node<T> tail;
    private int size;

    public LinkedDeque() {
        this.head = null;
        this.tail = null;
        this.size = 0;
    }

    public void addFirst(T item) {
        Node<T> newNode = new Node<>(item);
        if (isEmpty()) {
            head = tail = newNode;
        } else {
            newNode.next = head;
            head.prev = newNode;
            head = newNode;
        }
        size++;
    }

    public void addLast(T item) {
        Node<T> newNode = new Node<>(item);
        if (isEmpty()) {
            head = tail = newNode;
        } else {
            newNode.prev = tail;
            tail.next = newNode;
            tail = newNode;
        }
        size++;
    }

    public T removeFirst() {
        if (isEmpty()) throw new RuntimeException("Deque Underflow: 데이터가 존재하지 않습니다.");
        T data = head.data;
        head = head.next;
        if (head == null) tail = null;
        else head.prev = null;
        size--;
        return data;
    }

    public T removeLast() {
        if (isEmpty()) throw new RuntimeException("Deque Underflow: 데이터가 존재하지 않습니다.");
        T data = tail.data;
        tail = tail.prev;
        if (tail == null) head = null;
        else tail.next = null;
        size--;
        return data;
    }

    public boolean isEmpty() {
        return size == 0;
    }

    public int size() {
        return size;
    }
}
```

### 논리적 우아함과 구현의 마찰력

이러한 상호 치환 구현은 우리에게 중요한 교훈을 줍니다. 스택 두 개로 큐를 만드는 것은 실전에서도 '배치 처리(Batch Processing)'나 '읽기/쓰기 분리' 전략에서 유용하게 쓰일 수 있는 영리한 설계입니다. 하지만 큐로 스택을 만드는 것은 논리적 가능성을 증명할 뿐, 엔지니어링 관점에서는 불필요한 마찰을 발생시키는 비효율의 전형입니다.

> **자료구조 선택과 성능 비용**
> 
> 자료구조의 선택은 단순히 기능을 구현하는 것을 넘어, 시스템이 지불해야 할 성능 비용을 결정하는 행위입니다. 논리적으로 가능하다고 해서 그것이 반드시 올바른 구현은 아니며, 데이터의 흐름이 자료구조의 본질적 성향과 충돌할 때 발생하는 성능 저하를 고려해야 합니다.
{: .prompt-warning }

## 공간과 시간의 트레이드오프

우리는 종종 자료구조를 추상적인 논리의 성벽 안에서만 바라보곤 합니다. $O(1)$이라는 마법의 숫자가 찍히면 모든 고민이 끝난 것처럼 안심하지만, 실제 소프트웨어가 구동되는 하드웨어의 세계는 훨씬 더 냉혹하고 물리적입니다. 스택과 큐를 구현할 때 직면하는 최후의 선택은 결국 '연속된 공간이 주는 속도'와 '파편화된 공간이 주는 유연성' 사이의 균형점 찾기입니다.

### Cache Locality

배열 기반의 스택과 환형 큐가 연결 리스트 기반 구현보다 압도적으로 빠른 이유는 '캐시 지역성(Cache Locality)'에 있습니다. 현대의 CPU는 메모리에서 데이터를 가져올 때 딱 필요한 8바이트만 가져오지 않습니다. 주변의 데이터까지 한꺼번에 '캐시 라인'이라는 단위로 묶어 고속의 L1/L2 캐시에 올려둡니다.

배열은 데이터가 메모리상에 다닥다닥 붙어 있기 때문에, 한 번의 메모리 접근으로 다음 원소들까지 캐시에 로드될 확률이 매우 높습니다. 반면, 연결 리스트는 논리적으로는 바로 옆 노드일지라도 물리적 메모리 주소는 수 킬로미터 떨어진 곳에 배치될 수 있습니다. 이 경우 CPU는 매번 캐시 미스(Cache Miss)를 겪으며 느릿느릿한 메인 메모리로 손을 뻗어야 합니다. 알고리즘의 시간 복잡도가 동일한 $O(1)$일지라도, 실제 실행 시간(Wall-clock time)에서 수십 배의 격차가 벌어지는 근본적인 이유입니다.

### 정적 할당과 동적 할당의 기회비용

연결 리스트 기반의 구현은 '필요할 때만 메모리를 쓴다'는 점에서 우아해 보이지만, 여기에는 '할당자(Allocator)의 비용'이라는 숨겨진 세금이 붙습니다. 새로운 노드를 생성할 때마다 운영체제나 언어의 런타임에 메모리를 요청하는 행위는 생각보다 무거운 작업입니다. 특히 멀티 스레드 환경에서 메모리 할당을 위한 락(Lock) 경합이 발생하여 시스템 전체의 성능을 갉아먹기도 합니다.

반면 배열 기반 구현은 처음에 큰 덩어리를 예약하거나, 부족할 때만 가끔 확장(Resize)을 수행하므로 개별 연산에서의 오버헤드가 훨씬 적습니다. 다만, 예상보다 데이터가 적게 들어올 경우 미리 할당된 메모리가 낭비된다는 단점이 있습니다. 결국 우리는 **'메모리를 조금 낭비하더라도 예측 가능한 고성능을 얻을 것인가(Array)'**, 아니면 **'성능을 조금 희생하더라도 메모리를 알뜰하게 쓸 것인가(Linked List)'**를 결정해야 합니다.

| 구분               | 배열 기반 (Array-based)          | 연결 리스트 기반 (Linked List)      |
| :----------------- | :------------------------------- | :---------------------------------- |
| **캐시 지역성**    | **매우 높음** (연속된 메모리)    | **낮음** (메모리 파편화 가능성)     |
| **메모리 할당**    | 초기 1회 또는 희소한 재할당      | 매 원소 삽입 시 동적 할당           |
| **추가 오버헤드**  | 여분 공간 할당에 따른 낭비       | 포인터 저장용 추가 메모리 필요      |
| **최대 크기 제한** | 고정됨 (Resize 시 큰 비용 발생)  | 논리적으로 무제한 (Heap 가용 범위)  |
| **최적의 상황**    | 크기가 예측 가능한 고성능 시스템 | 크기 변동이 심하고 메모리 절약 우선 |


## Summary

스택과 큐는 단순히 데이터를 저장하는 바구니가 아니라, 알고리즘에 '시간'과 '질서'의 관념을 부여하는 도구입니다. 스택은 현재의 맥락을 보존하며 심연(Recursion)으로 내려가기 위한 생명줄이며, 큐는 폭주하는 요청들 사이에서 공정함을 유지하며 흐름을 조율하는 관제탑입니다. 엔지니어로서 우리가 해야 할 일은 추상적인 $O(1)$의 수식 뒤에 숨겨진 메모리 레이아웃, 캐시 효율성, 그리고 분할 상환의 비용을 꿰뚫어 보고 상황에 가장 적합한 물리적 형태를 선택하는 것입니다. 완벽한 자료구조는 없으며, 오직 특정 상황에서의 '최선의 트레이드오프'만이 존재할 뿐입니다.

## References

* [[Interview Cake] Dynamic Array Amortized Analysis](https://www.interviewcake.com/concept/java/dynamic-array-amortized-analysis)
* [[Stanford CS166] Amortized Analysis & The Two-Stack Queue](https://web.stanford.edu/class/archive/cs/cs166/cs166.1256/lectures/08/Condensed%20Sildes.pdf)
* [[Runestone Academy] Array-Based vs Linked Queues](https://runestone.academy/ns/books/published/bridgesds/sec-ArrayvsQueues.html)
* [[Hero Vired] Circular Queue Implementation & Modular Arithmetic](https://herovired.com/home/learning-hub/topics/circular-queue-in-data-structure)