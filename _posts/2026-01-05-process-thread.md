---
layout: post
title: "[OS] Process와 Thread"
date: 2026-01-05 20:59 +0900
categories:
- Computer Science
- Operating System & Hardware
tags:
- Process
- Thread
image:
    path: /assets/img/2026-01-05-23-14-33.png
---

정적인 파일에 불과한 프로그램이 메모리에 올라와 생명력을 얻는 순간, 우리는 이를 프로세스라 부릅니다. 운영체제는 가상화라는 마법을 통해 각 프로세스에게 독립된 영토를 보장하고, CPU는 찰나의 순간마다 이들을 오가며 연산을 수행합니다. 하지만 이러한 추상화의 이면에는 컨텍스트 스위칭이라는 물리적 마찰과 자원 공유의 딜레마가 숨어 있습니다. 프로세스와 스레드의 동작 원리를 깊이 이해하는 것은 복잡한 시스템 환경에서 병목을 진단하고 최적화할 수 있는 엔지니어의 기초 체력을 기르는 과정입니다.

## 🧩 Process

"실행 중인 프로그램"이라는 정의는 단순하지만, 그 이면에 숨겨진 커널의 고군분투는 결코 단순하지 않습니다. 디스크에 잠들어 있던 정적인 바이너리가 메모리에 올라와 '생명력'을 얻는 순간, 운영체제는 이 실체 없는 존재에게 물리적 자원을 할당하고 격리된 영토를 보장해 주어야 합니다.

### 실행 중인 프로그램의 실체

우리는 흔히 프로그램을 프로세스와 혼용하곤 하지만, 둘 사이에는 명확한 물리적 경계가 존재합니다. 프로그램이 디스크에 저장된 '설계도(Static Object)'라면, 프로세스는 그 설계도가 메모리에 적재되어 CPU라는 엔진을 점유하기 시작한 '활동적인 실체(Dynamic Entity)'입니다. 

운영체제는 **가상화(Virtualizing)**라는 강력한 추상화를 통해 프로세스에게 "이 컴퓨터의 자원을 나 혼자 쓰고 있다"는 환상을 심어줍니다. CPU는 시분할(Time-sharing)을 통해 여러 프로세스를 번갈아 수행하고, 메모리는 가상 주소 공간을 통해 마치 광활한 단일 영토처럼 보이게 만듭니다. 이러한 추상화 덕분에 우리는 복잡한 하드웨어 제어 로직 없이도 독립적인 실행 문맥을 유지할 수 있습니다.

### Process Control Block (PCB)

프로세스가 실행 문맥을 유지하며 자원을 점유하기 위해서는 커널 어딘가에 이 정보를 기록할 '장부'가 필요합니다. 이것이 바로 **PCB(Process Control Block)**입니다.

커널 주소 공간 내에 위치한 PCB는 프로세스의 모든 것을 명세합니다. 고유 식별자인 PID부터 현재 어느 명령어를 수행 중인지 알려주는 PC(Program Counter), 그리고 계산 결과가 담긴 각종 레지스터 값들이 이 장부에 꼼꼼히 기록됩니다.

> **Context Switch의 본질**
> 
> CPU가 다른 프로세스로 고개를 돌릴 때, 커널은 현재 수행 중인 상태를 해당 프로세스의 PCB(정확히는 스택 공간)에 백업합니다. 이후 새로 실행할 프로세스의 PCB에서 이전 상태를 복원하여 CPU 레지스터에 로드합니다. 이 일련의 과정을 컨텍스트 스위칭이라 부르며, 이는 운영체제가 다중 작업을 지원하기 위해 지불하는 가장 핵심적인 비용입니다.
{: .prompt-info }

### Process Memory Layout

프로세스는 커널로부터 **가상 주소 공간(Virtual Address Space, VAS)**이라는 독립된 영토를 부여받습니다. 이 공간은 크게 네 가지 영역으로 설계됩니다.

1. **Text (Code)**: 작성한 소스 코드가 기계어로 변환되어 저장되는 읽기 전용 공간입니다. 로직이 변조되는 것을 방지하기 위해 엄격한 권한 관리가 이루어집니다.
2. **Data & BSS**: 전역 변수와 정적 변수가 거주합니다. 초기값이 있으면 Data, 없으면 BSS(Block Started by Symbol) 영역에 할당되어 메모리 효율을 극대화합니다.
3. **Heap**: 개발자가 런타임에 필요에 따라 동적으로 확장하는 영토입니다. 자바의 객체들이 바로 이곳에 생성됩니다.
4. **Stack**: 함수 호출의 기록과 지역 변수가 저장되는 LIFO 구조의 공간입니다.

흥미로운 점은 Heap과 Stack의 배치입니다. Heap은 낮은 주소에서 위로, Stack은 높은 주소에서 아래로 서로를 마주 보며 성장하도록 설계되어 공간 활용도를 높입니다. 이때 두 영역이 충돌하여 데이터가 오염되는 것을 방지하기 위해 커널은 그 경계에 **Guard Page**라는 접근 불가능한 장벽을 세워 안전성을 확보합니다.

### 프로세스 생애 주기와 상태 전이

프로세스의 삶은 정적인 단계에 머물지 않고 끊임없이 전이됩니다.

* **New → Ready**: 프로그램이 메모리에 로드되고 PCB가 생성되면 '준비' 상태가 됩니다.
* **Ready → Running**: 스케줄러에 의해 선택(Dispatch)받아 CPU를 점유하면 '실행' 상태가 됩니다.
* **Running → Waiting**: I/O 요청과 같은 시스템 콜을 호출하면 CPU를 반납하고 '대기' 상태로 들어갑니다. 작업이 완료되면 다시 Ready 상태로 돌아가 차례를 기다립니다.
* **Terminated**: 모든 임무를 마치면 자원을 반납하고 소멸합니다.

이 과정에서 **타이머 인터럽트**는 특정 프로세스가 CPU를 독점하지 못하도록 강제로 선점(Preemption)하여 다른 프로세스에게 기회를 줍니다. 이는 민주적이고 공정한 시스템 운영의 근간이 됩니다.

### 리눅스 커널의 프로세스 관리

리눅스 커널 내부에서 프로세스는 `task_struct`라는 거대한 구조체로 관리됩니다. 그리고 이 구조체 안에는 메모리 지도를 관리하는 `mm_struct`가 존재합니다.

```c
/* include/linux/sched.h */
struct task_struct {
    volatile long state;    /* 프로세스 상태 */
    struct mm_struct *mm;   /* 프로세스 메모리 기술자 */
    pid_t pid;
    struct task_struct *parent;
    /* ... 수많은 필드들 ... */
};

/* include/linux/mm_types.h */
struct mm_struct {
    struct vm_area_struct *mmap;    /* VMA 리스트 */
    unsigned long start_code, end_code, start_data, end_data;
    unsigned long start_brk, brk, start_stack;
    /* ... */
};
```

리눅스에서 새로운 프로세스를 만드는 가장 우아한 방식은 `fork()`입니다. 하지만 수 GB의 메모리를 가진 프로세스를 그대로 복사하는 것은 너무나 무거운 작업입니다. 이를 해결하기 위해 커널은 **Copy-on-Write (CoW)** 기술을 사용합니다. 부모와 자식이 같은 물리 메모리 페이지를 공유하다가, 실제로 어느 한쪽에서 쓰기(Write) 작업이 발생할 때만 해당 페이지를 복제하는 방식입니다. 이를 통해 프로세스 생성 비용을 비약적으로 낮출 수 있습니다.

### [실습] 프로세스 가시화 및 분석

이론으로만 접하던 프로세스의 영토를 직접 확인하는 방법은 간단합니다. 리눅스의 `/proc` 파일 시스템은 커널의 투명한 장부 역할을 합니다.

```bash
# 특정 프로세스의 메모리 맵 확인 (PID가 1234일 때)
$ pmap -x 1234

# 커널이 관리하는 프로세스의 상세 상태 장부 확인
$ cat /proc/1234/status

# 주요 지표 예시:
# VmSize: 가상 메모리 총량
# VmRSS: 실제 물리 메모리에 상주하는 양 (Resident Set Size)
# Threads: 해당 프로세스 내의 스레드 개수
```

`pmap` 명령어를 통해 출력된 메모리 맵에서 우리는 앞서 배운 VAS의 각 세그먼트 주소와 권한(read, write, exec)을 직접 목격할 수 있습니다. 이는 추상이 아닌 물리적 실체로서의 프로세스를 이해하는 첫걸음이 됩니다.

## 🧩 Thread

프로세스가 운영체제로부터 부여받은 '독립된 요새'라면, 스레드는 그 요새 안에서 같은 자원을 공유하며 효율적으로 움직이는 '경량화된 실행 단위'입니다. 과거에는 하나의 프로세스가 하나의 일만 수행했지만, 현대의 복잡한 애플리케이션은 동시에 여러 흐름을 처리해야 했고, 이를 위해 탄생한 것이 바로 스레드입니다.

### Lightweight Process

스레드는 프로세스의 자원을 공유하면서도 각자 독립적인 실행 문맥을 가집니다. 프로세스를 새로 생성하는 것(fork)은 주소 공간 전체를 복사해야 하므로 비용이 매우 비싸지만, 스레드는 기존 프로세스의 영토를 그대로 사용하기에 생성 및 전환 속도가 압도적으로 빠릅니다.

이러한 특성 때문에 스레드는 **Lightweight Process(LWP)**라고도 불립니다. 하지만 자원을 공유한다는 것은 양날의 검과 같습니다. 효율성은 극대화되지만, 여러 스레드가 동시에 같은 메모리에 접근할 때 발생하는 데이터 레이스(Race Condition)는 개발자가 평생 짊어져야 할 숙명이 되었습니다.

### Thread Control Block (TCB)

프로세스에 PCB가 있듯, 스레드에게도 자신의 상태를 기록할 장부가 필요합니다. 이것이 **TCB(Thread Control Block)**입니다. 

TCB는 스레드가 현재 어디까지 실행했는지(PC), 연산 중인 값은 무엇인지(Register Set)를 기록합니다. 여기서 중요한 점은 **공유 영역과 독립 영역의 구분**입니다.

* **공유 영역**: Code, Data, Heap 세그먼트와 열려 있는 파일(Files), 시그널(Signals) 등 프로세스 수준의 자원은 모든 스레드가 함께 점유합니다.
* **독립 영역**: 각 스레드는 자신만의 호출 기록을 남기기 위한 **Stack**과 프로그램 카운터(PC)를 가집니다.

### 멀티스레딩 모델과 실행 아키텍처

하나의 프로세스 주소 공간 안에서 여러 스레드가 공존하기 위해, 커널은 프로세스의 스택 영역을 잘게 쪼개어 각 스레드에게 할당합니다. 이를 **멀티 스택(Multi-stack)** 구조라고 합니다.

또한, 유저 레벨의 스레드와 커널 레벨의 태스크를 어떻게 매핑하느냐에 따라 성능 특성이 달라집니다.
1. **1:1 모델 (Kernel-level)**: 자바의 전통적인 스레드 모델로, 유저 스레드 하나가 실제 커널 태스크 하나와 매핑됩니다. 구현이 단순하고 멀티코어를 활용하기 좋지만, 스레드 생성 비용이 크다는 단점이 있습니다.
2. **M:N 모델 (Hybrid)**: 여러 유저 스레드를 적은 수의 커널 스레드 위에서 스케줄링합니다. 고도의 기술력이 필요하지만, 최근 Java 21의 가상 스레드(Virtual Threads)가 이 방식을 현대적으로 재해석하여 도입했습니다.

### 리눅스와 자바의 구현체 분석

기술적으로 흥미로운 사실은, 리눅스 커널 입장에서 프로세스와 스레드는 본질적으로 큰 차이가 없다는 점입니다. 리눅스는 모든 실행 단위를 `task_struct`로 다루며, 오직 `clone()` 시스템 콜에 전달되는 플래그 값에 따라 이를 프로세스로 부를지 스레드로 부를지 결정합니다.

```c
/* Eli Bendersky의 분석에 따르면, 리눅스 스레드는 CLONE_VM 등의 플래그를 통해 생성됨 */

#define _GNU_SOURCE
#include <sched.h>
#include <stdio.h>

/* 스레드 생성 시 전달되는 주요 플래그 */
// CLONE_VM: 부모와 자식 간 메모리 주소 공간 공유
// CLONE_FS: 파일 시스템 정보 공유
// CLONE_FILES: 파일 서술자 테이블 공유
// CLONE_SIGHAND: 시그널 핸들러 공유

int flags = CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND;
// pid = clone(child_func, stack_top, flags, arg);
```

자바의 `java.lang.Thread`는 내부적으로 이 `clone()` 또는 `pthread_create`를 호출하여 OS 네이티브 스레드를 생성합니다. 즉, 우리가 자바에서 스레드를 하나 만드는 행위는 OS 커널에게 값비싼 자원을 요청하는 무거운 작업임을 인지해야 합니다.

> **Thread Local Storage (TLS)**
> 
> 공유 자원 속에서도 스레드만의 비밀 금고가 필요할 때가 있습니다. 이를 TLS라고 하며, Java에서는 `ThreadLocal` 클래스를 통해 구현됩니다. 하지만 스레드 풀 환경에서는 이 금고를 비우지 않으면 이전 데이터가 다음 작업으로 전이되는 사고가 발생할 수 있으니 주의가 필요합니다.
{: .prompt-warning }

### [실습] 런타임 스레드 분석

현재 내 애플리케이션에서 스레드가 어떻게 동작하는지 확인하는 가장 강력한 도구는 `jstack`과 `strace`입니다.

```bash
# 1. 자바 프로세스의 스레드 덤프 추출
$ jstack <PID> > thread_dump.txt

# thread_dump.txt 예시:
# "Thread-0" #20 prio=5 os_prio=0 cpu=1.52ms elapsed=10.21s tid=0x0000 nid=0x1234 waiting on condition [0x...]
#    java.lang.Thread.State: WAITING (parking)

# 2. 자바 프로세스 생성 시 발생하는 시스템 콜 추적 (clone 포착)
$ strace -f -e clone java MyApp
# [pid  1234] clone(child_stack=0x..., flags=CLONE_VM|CLONE_FS|CLONE_FILES|...) = 1235
```

`jstack`을 통해 각 스레드가 어떤 Lock을 기다리고 있는지(BLOCKED), 혹은 어떤 코드를 실행 중인지(RUNNABLE)를 파악함으로써 병목 지점을 찾아낼 수 있습니다. 또한 `strace`를 통해 자바 애플리케이션이 실제 OS 레벨에서 `clone()`을 호출하는 찰나를 포착해 보면, 추상화 뒤에 숨겨진 시스템의 실체를 실감할 수 있습니다.

## Context Switch와 하드웨어 마찰

우리는 종종 컨텍스트 스위칭을 운영체제가 제공하는 당연하고 저렴한 서비스처럼 여깁니다. 하지만 시스템의 처리량(Throughput)이 임계치에 도달했을 때, 진정한 병목은 소프트웨어의 알고리즘이 아니라 하드웨어 내부에서 발생하는 '물리적 비명'에 있습니다.

### CR3 레지스터 교체와 TLB Flush

프로세스가 바뀐다는 것은 단순히 레지스터 값을 갈아 끼우는 것 이상의 의미를 가집니다. CPU가 메모리에 접근하기 위해 참조하는 **가상 주소 공간(VAS)**의 지도 자체가 바뀌는 것이기 때문입니다.

x86 아키텍처에서 커널은 프로세스 전환 시 **CR3 레지스터**(Control Register 3)를 새로운 프로세스의 페이지 디렉터리 주소로 교체합니다. 이 순간, CPU 내부의 초고속 주소 변환 캐시인 **TLB(Translation Lookaside Buffer)**는 일대 혼란에 빠집니다. 이전 프로세스의 주소 변환 정보는 더 이상 유효하지 않게 되며, 커널은 대개 TLB를 비워버리는(Flush) 선택을 합니다.

> **ASID (Address Space Identifier)의 완화**
> 
> 최신 프로세서들은 PCID(Process Context Identifier) 혹은 ASID라는 태그를 TLB 항목에 붙여, 프로세스가 바뀌어도 자신의 정보를 유지하게 함으로써 Flush 오버헤드를 줄이려 노력합니다. 하지만 태그 공간의 한계로 인해 잦은 스위칭 환경에서의 성능 저하는 여전히 피할 수 없는 현실입니다.
{: .prompt-info }

### Cache Pollution과 Memory Stall

더 치명적인 문제는 CPU 캐시에서 발생합니다. 현대의 CPU는 메모리보다 훨씬 빠른 속도를 내기 위해 L1, L2, L3 캐시에 데이터를 미리 채워두고 재사용합니다. 이를 데이터 지역성(Locality)이라 부릅니다.

하지만 컨텍스트 스위칭이 발생하여 새로운 프로세스가 CPU를 점유하면, 이 프로세스는 자신의 업무를 위해 기존 캐시에 담겨 있던 '남의 데이터'를 밀어내고 자신의 데이터를 채우기 시작합니다. 이를 **캐시 오염(Cache Pollution)**이라 합니다.

1. **Cold Cache**: 새로 들어온 프로세스는 캐시 적중률(Hit Rate)이 극도로 낮은 상태에서 실행을 시작합니다.
2. **Memory Stall**: 캐시 미스가 발생할 때마다 CPU는 수백 사이클 동안 아무 일도 하지 못한 채 메인 메모리(RAM)로부터 데이터가 오기만을 기다리며 멈춰 서게 됩니다.

결국 잦은 컨텍스트 스위칭은 CPU가 실제 연산을 수행하는 시간보다 데이터를 기다리며 멍하게 있는 시간을 늘려, 전체 시스템의 에너지를 소모시키는 주범이 됩니다.

### 프로세스 vs 스레드 스위칭의 비용 차이

흔히 스레드 스위칭이 프로세스 스위칭보다 빠르다고 말하는 이유는 바로 이 **'하드웨어 마찰'**의 유무 때문입니다. 스레드들은 같은 주소 공간을 공유하므로, 스레드 간 전환 시에는 CR3 레지스터를 교체할 필요가 없고 TLB를 비울 이유도 없습니다.

물론 스레드 전환 시에도 CPU 레지스터 값을 복원하는 오버헤드와 캐시의 일부가 새로운 스레드의 스택 데이터로 덮여 써지는 현상은 발생합니다. 하지만 메모리 지도(Page Table)를 통째로 갈아치워야 하는 프로세스 전환에 비하면, 하드웨어가 느끼는 피로감은 훨씬 덜합니다.

```c
/* * 실제 유저 모드에서는 실행할 수 없으며, 커널 모드 권한이 필요합니다.
 * 컨텍스트 스위칭 시 커널 내부에서 일어나는 동작을 개념적으로 보여줍니다. 
 */

unsigned long long read_cr3(void) {
    unsigned long long val;
    // CR3 레지스터의 값을 읽어와 변수 val에 저장
    __asm__ __volatile__(
        "mov %%cr3, %0"
        : "=r" (val)
    );
    return val;
}

void write_cr3(unsigned long long val) {
    // 새로운 페이지 디렉터리 주소를 CR3에 쓰기
    // 이 작업 직후 TLB의 비태그(non-tagged) 항목들은 무효화(Flush)됩니다.
    __asm__ __volatile__(
        "mov %0, %%cr3"
        : : "r" (val)
    );
}
```

> **시니어의 관점**
> 
> 고성능 서버 애플리케이션을 설계할 때, 무조건 스레드 수를 늘리는 것이 답이 아닌 이유가 여기에 있습니다. 스레드가 CPU 코어 수보다 과도하게 많아지면, 하드웨어는 유의미한 연산을 처리하기보다 캐시를 비우고 주소 지도를 바꾸는 '살림살이 교체'에 더 많은 시간을 쓰게 됩니다.
{: .prompt-tip }

### [실습] 하드웨어 마찰의 가시화

리눅스 환경에서는 `perf` 도구를 통해 이러한 하드웨어 수준의 마찰을 수치로 확인할 수 있습니다.

```bash
# 1. 특정 애플리케이션의 컨텍스트 스위치와 캐시 미스 동시 측정 (5초간)
$ perf stat -e context-switches,cpu-migrations,page-faults,cache-misses,L1-dcache-load-misses -p <PID> sleep 5

# 결과 예시 분석:
# context-switches 수치가 높을 때 cache-misses 비율이 동반 상승한다면
# 과도한 스레드 경합으로 인한 성능 저하를 의심해야 합니다.

# 2. 시스템 전체에서 컨텍스트 스위칭이 빈번한 지점 프로파일링
$ perf record -e context-switches -a -g -- sleep 10
$ perf report
```

`context-switches` 수치와 `cache-misses`, `L1-dcache-load-misses`의 상관관계를 추적해 보면, 애플리케이션의 성능 저하가 단순히 코드의 비효율 때문인지, 아니면 과도한 컨텍스트 스위칭으로 인한 하드웨어의 한계 때문인지 명확히 진단할 수 있습니다.

## Project Loom: Virtual Threads의 혁신

수십 년간 자바 백엔드 개발자들을 지배해온 철칙은 "요청당 스레드 하나(Thread-per-request)"였습니다. 하지만 클라우드 네이티브 시대에 접어들어 수만 개의 동시 접속을 처리해야 하는 상황이 되자, 우리가 믿어왔던 이 모델은 거대한 벽에 부딪혔습니다. Project Loom은 이 벽을 허물기 위해 자바 런타임에 **가상 스레드(Virtual Threads)**라는 혁신을 도입했습니다.

### 플랫폼 스레드 매핑 모델의 임계점

전통적인 자바 스레드(Platform Thread)는 OS 커널 스레드와 1:1로 매핑됩니다. 이 구조는 단순하고 강력하지만, 두 가지 치명적인 비용을 발생시킵니다.

1. **메모리 점유(Footprint)**: OS 스레드는 생성 시 약 1MB 정도의 스택 메모리를 예약합니다. 만약 수만 개의 스레드를 만든다면 메모리는 순식간에 고갈됩니다.
2. **스케줄링 부하**: 앞서 살펴본 것처럼, 커널 레벨의 컨텍스트 스위칭은 TLB Flush와 캐시 마찰을 유발하는 값비싼 연산입니다.

결국 "동시성(Concurrency)의 한계가 곧 OS 스레드의 한계"가 되는 상황이 벌어진 것입니다. 이를 해결하기 위해 리액티브 프로그래밍(WebFlux 등)이 등장했지만, 비동기 논블로킹 코드는 디버깅이 어렵고 가독성이 떨어진다는 대가를 치러야 했습니다.

### 가상 스레드(Virtual Threads)의 등장

가상 스레드는 OS 커널이 아닌 **JVM 스케줄러**가 관리하는 경량 스레드입니다. 수백 개에 불과한 OS 스레드(Carrier Threads) 위에서 수백만 개의 가상 스레드가 마치 구름처럼 떠다니며 실행됩니다.

핵심은 가상 스레드가 I/O 작업으로 인해 차단(Blocking)될 때의 동작입니다. 기존 모델에서는 커널 스레드 자체가 멈춰버렸지만, 가상 스레드 환경에서는 JVM이 차단된 가상 스레드를 '잠시 내려놓고(Unmount)' 다른 가상 스레드를 그 자리에 올립니다. 덕분에 실제 OS 스레드는 쉬지 않고 계속해서 유의미한 연산을 수행할 수 있게 됩니다.

> **M:N 스케줄링의 부활**
> 
> 가상 스레드는 수많은(M) 유저 레벨 스레드를 적은 수의(N) 커널 레벨 스레드에 매핑하는 M:N 모델을 현대적으로 구현한 것입니다. 이를 통해 개발자는 익숙한 동기(Blocking) 방식의 코드를 작성하면서도, 리액티브 시스템에 준하는 고성능 처리량을 얻을 수 있습니다.
{: .prompt-tip }

### Continuation: 힙(Heap) 영역을 이용한 실행 문맥 스위칭

가상 스레드가 '내려놓아질 때' 현재의 실행 상태(Stack Frame)는 어디로 갈까요? 여기서 **Continuation** 메커니즘이 등장합니다.

가상 스레드가 차단되면 JVM은 스택에 쌓여있던 실행 문맥을 **Java 힙(Heap) 영역**으로 통째로 복사하여 저장합니다. 그리고 나중에 I/O 작업이 완료되면 힙에 있던 데이터를 다시 OS 스레드의 실제 스택으로 복원(Mount)하여 중단된 지점부터 실행을 재개합니다.

커널의 개입 없이 메모리 복사만으로 문맥 교환이 일어나기 때문에, 컨텍스트 스위칭 비용이 획기적으로 줄어듭니다. 스레드 100만 개를 생성해도 수 GB의 메모리만으로 충분한 이유가 바로 여기에 있습니다.

### [실습] 플랫폼 스레드 vs 가상 스레드 확장성 테스트

가상 스레드의 위력은 간단한 부하 테스트만으로도 체감할 수 있습니다.

```java
import java.time.Duration;
import java.util.concurrent.Executors;
import java.util.stream.IntStream;

public class VirtualThreadTest {
    public static void main(String[] args) {
        long start = System.currentTimeMillis();

        // 가상 스레드 전용 Executor 생성
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            IntStream.range(0, 1_000_000).forEach(i -> {
                executor.submit(() -> {
                    // 가상의 I/O 작업 시뮬레이션
                    Thread.sleep(Duration.ofSeconds(1));
                    return i;
                });
            });
        } // try-with-resources에 의해 모든 작업 완료 대기

        long end = System.currentTimeMillis();
        System.out.println("걸린 시간: " + (end - start) + "ms");
    }
}
```

전통적인 방식이라면 `OutOfMemoryError`로 프로세스가 즉시 종료되었겠지만, 가상 스레드는 수 초 내에 100만 개의 작업을 가볍게 완수합니다. 이는 더 이상 스레드가 '비싼 자원'이 아니며, 필요할 때마다 마음껏 생성하고 버리는 **'소모성 객체'**가 되었음을 의미합니다.

> **주의: ThreadLocal 오염**
> 
> 가상 스레드는 스레드 풀(Pool)을 만들지 않고 매번 새로 생성하는 것을 권장합니다. 하지만 기존 라이브러리들이 `ThreadLocal`에 무거운 객체를 캐싱하고 있다면, 수백만 개의 가상 스레드가 각각 그 객체를 가지려다 메모리 폭발이 일어날 수 있으니 주의 깊은 설계가 필요합니다.
{: .prompt-warning }

## 자원 고갈과 오염

운영체제와 언어 런타임이 제공하는 세련된 추상화는 개발자를 복잡한 하드웨어 제어로부터 해방시켜 주었지만, 역설적으로 그 추상화 아래에서 벌어지는 자원 누수와 오염에 무감각하게 만들기도 합니다. 특히 대규모 트래픽을 처리하는 백엔드 환경에서는 작은 관리 소홀이 시스템 전체의 불능(Incapacitation)으로 이어지는 치명적인 함정이 됩니다.

### Zombie & Orphan Process: 종료 코드 미수거에 따른 커널 장부 잔류

프로세스가 실행을 마치고 종료되면 모든 자원이 즉시 회수될 것 같지만, 커널 수준에서는 그렇지 않습니다. 프로세스가 종료된 후에도 커널은 해당 프로세스의 종료 상태(Exit Status)를 부모 프로세스가 확인할 때까지 최소한의 정보인 `task_struct`를 메모리에 유지합니다.

1. **Zombie Process**: 자식 프로세스가 종료되었으나 부모가 `wait()` 또는 `waitpid()` 시스템 콜을 호출하여 종료 코드를 수거하지 않은 상태입니다. 실질적인 메모리 점유는 적지만, 커널의 프로세스 식별자(PID) 공간을 차지하여 결국 새로운 프로세스 생성을 차단하는 **PID 고갈** 사태를 초래합니다.
2. **Orphan Process**: 부모 프로세스가 먼저 종료되어 버린 자식들입니다. 다행히 현대 운영체제에서는 `init` 프로세스(PID 1)가 이들을 입양하여 종료 코드를 대신 수거해 주지만, 설계되지 않은 부모 프로세스의 갑작스러운 종료는 시스템의 논리적 흐름을 파괴할 수 있습니다.

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

/**
 * 부모 프로세스가 wait()을 호출하지 않고 잠들면, 
 * 먼저 종료된 자식은 Zombie(defunct) 상태가 됩니다.
 */
int main() {
    pid_t pid = fork();

    if (pid > 0) {
        // 부모 프로세스: 자식의 종료를 수거하지 않고 60초간 대기
        printf("Parent process (PID: %d) is sleeping...\n", getpid());
        sleep(60); 
    } else if (pid == 0) {
        // 자식 프로세스: 즉시 종료
        printf("Child process (PID: %d) is exiting...\n", getpid());
        exit(0);
    }

    return 0;
}
```

### ThreadPool 환경에서의 ThreadLocal 데이터 오염

자바 백엔드 엔지니어에게 가장 위험한 함정 중 하나는 `ThreadLocal`입니다. 스레드별로 독립적인 저장소를 제공하는 이 편리한 도구는 **스레드 풀(Thread Pool)** 환경과 만나는 순간 시한폭탄으로 변합니다.

전통적인 자바 웹 서버는 스레드를 매번 생성하지 않고 풀에 담아 재사용합니다. 만약 특정 요청을 처리하던 스레드가 `ThreadLocal`에 사용자 인증 정보나 트랜잭션 데이터를 남겨둔 채 풀로 돌아가고, 이를 제대로 비우지(`remove()`) 않는다면 어떻게 될까요?

* **데이터 오염**: 다음 요청을 배정받은 전혀 다른 사용자가 이전 사용자의 데이터를 참조하게 되는 보안 사고가 발생합니다.
* **메모리 누수**: 스레드가 살아있는 한 `ThreadLocal`에 담긴 객체는 가비지 컬렉션(GC)의 대상이 되지 않아 서서히 힙 영역을 잠식합니다.

> **시니어의 트러블슈팅**
> 
> `ThreadLocal` 오염은 재현이 어렵고 간헐적으로 발생하여 디버깅 난이도가 매우 높습니다. 따라서 비즈니스 로직의 마지막 단계(대개 Filter나 Interceptor)에서 반드시 `try-finally` 블록을 통해 명시적으로 `remove()`를 호출하는 설계를 강제해야 합니다.
{: .prompt-warning }

```java
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

/**
 * 부모 프로세스가 wait()을 호출하지 않고 잠들면, 
 * 먼저 종료된 자식은 Zombie(defunct) 상태가 됩니다.
 */
int main() {
    pid_t pid = fork();

    if (pid > 0) {
        // 부모 프로세스: 자식의 종료를 수거하지 않고 60초간 대기
        printf("Parent process (PID: %d) is sleeping...\n", getpid());
        sleep(60); 
    } else if (pid == 0) {
        // 자식 프로세스: 즉시 종료
        printf("Child process (PID: %d) is exiting...\n", getpid());
        exit(0);
    }

    return 0;
}
```

### Stack vs Heap의 실체: Reference Trap

우리는 지역 변수는 스택에, 객체는 힙에 저장된다고 배웁니다. 하지만 스택에 위치한 '참조 변수'가 힙의 '가변 객체'를 가리키고, 그 객체가 여러 스레드에 의해 공유될 때 발생하는 설계적 결함은 찾아내기 어렵습니다. 

특히 람다식이나 익명 클래스 내에서 외부 지역 변수를 캡처할 때 발생하는 의도치 않은 객체 생명주기 연장은 메모리 프로파일링 도구 없이는 발견하기 힘든 자원 고갈의 원인이 됩니다.

### [실습] 시스템 자원 한계 측정 및 모니터링

엔지니어는 이론에 의존하기보다 실제 시스템의 한계를 수치로 파악하고 있어야 합니다.

```bash
# 1. 현재 쉘에서 생성 가능한 최대 프로세스(스레드 포함) 수 확인
$ ulimit -u

# 2. 프로세스당 오픈 가능한 파일 서술자 수 확인 (DB 커넥션 등에 영향)
$ ulimit -n

# 3. 커널 전체의 최대 PID 값 확인
$ cat /proc/sys/kernel/pid_max

# 4. 좀비 프로세스 확인 (S 또는 STAT 열에 'Z' 표시)
$ ps -efl | grep defunct
```

`ulimit` 설정을 통해 프로세스가 생성할 수 있는 최대 스레드 수나 열 수 있는 파일 서술자(File Descriptor)의 한계를 확인하고, 애플리케이션의 설정값과 정합성을 맞추는 작업은 장애 방지의 기본입니다.

## Summary

프로세스와 스레드의 본질을 이해하는 것은 단순히 실행 단위를 구분하는 것을 넘어, **운영체제가 자원을 격리하고 공유하는 방식**을 이해하는 과정입니다.

| 구분            | Process                                   | Thread                                       |
| :-------------- | :---------------------------------------- | :------------------------------------------- |
| **본질**        | 자원 소유의 단위 (Resource Ownership)     | 실행의 단위 (Unit of Execution)              |
| **메모리**      | 독립된 가상 주소 공간 (VAS)               | 프로세스 내 Stack 제외 나머지 공유           |
| **스위칭 비용** | **High** (CR3 교체, TLB Flush 발생)       | **Low** (주소 공간 유지, 캐시 효율 높음)     |
| **데이터 공유** | IPC (Pipe, Shared Memory 등) 필요         | 공유 메모리 직접 접근 (경합 주의)            |
| **실패 영향**   | 자식의 실패가 부모에게 직접 전이되지 않음 | 한 스레드의 예외가 프로세스 전체를 종료 가능 |

### 핵심 엔지니어링 체크포인트
- **격리와 성능의 트레이드오프**: 프로세스는 완벽한 격리를 제공하지만 컨텍스트 스위칭 시 하드웨어 마찰(Cache Pollution)을 피할 수 없습니다.
- **스레드 모델의 진화**: 자바는 1:1 네이티브 매핑의 한계를 극복하기 위해 Project Loom(Virtual Threads)을 도입했으며, 이는 Continuation 메커니즘을 통해 커널이 아닌 힙(Heap)에서 문맥을 관리합니다.
- **자원 관리의 책임**: Zombie 프로세스 방지를 위한 종료 코드 수거와 ThreadPool 환경에서의 ThreadLocal 클린업은 시스템 안정성을 결정짓는 엔지니어의 기초 체력입니다.

## References

* **[OSTEP]** Operating Systems: Three Easy Pieces - [Process Concept & VM Paging](https://pages.cs.wisc.edu/~remzi/OSTEP/)
* **[ManyButFinite]** [Anatomy of a Program in Memory](https://manybutfinite.com/post/anatomy-of-a-program-in-memory/)
* **[Eli Bendersky]** [Launching Linux threads and processes with clone](https://eli.thegreenplace.net/2018/launching-linux-threads-and-processes-with-clone/)
* **[Frank Denneman]** [Memory Configuration & Scalability Blog Series](https://frankdenneman.nl/2015/02/18/memory-configuration-scalability-blog-series/)
* **[OpenJDK]** [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)