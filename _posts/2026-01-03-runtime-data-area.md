---
layout: post
title: "[JVM Internal] Runtime Data Area"
date: 2026-01-03 07:35 +0900
categories:
- Java Ecosystem
- JVM Internal
tags:
- JVM
- Runtime Data Area
image:
  path: /assets/img/2026-01-03-12-32-12.png
---

JVM은 프로그램 실행 중에 사용되는 다양한 데이터 영역을 정의합니다. 이러한 데이터 영역 중 일부는 JVM 시작 시 생성되어 JVM이 종료될 때만 소멸(Shared)됩니다. 다른 데이터 영역은 스레드별로 관리되어 스레드가 생성될 때 생성되고 스레드가 종료될 때 소멸(Per-Thread)됩니다.

## 🧩 Per-Thread Data Areas (스레드 별 격리 공간)

### 1. PC Register

CPU의 물리적 레지스터가 아닌, OS 스레드 내의 가상 주소 카운터입니다.

JVM은 동시에 여러 실행 스레드를 지원할 수 있으며, 각 스레드는 자체적인 PC Register를 가지고 있습니다. PC Register는 현재 수행 중인 JVM 명령(Opcode)의 주소를 가리킵니다.

#### Native 메서드 실행 시의 특이점

만약 자바 스레드가 `native` 메서드(C/C++)를 호출하여 실행 중일 때, PC Register의 값은 Undefined 상태가 됩니다. 왜냐하면 Native 코드는 JVM의 통제 범위를 벗어난 OS 레벨의 명령어를 수행하기 때문입니다.

> PC는 플랫폼에 따라 워드 크기가 달라지지만, JVM 내부적으로는 추상화되어 있습니다.
{: .prompt-info }

### 2. JVM Stack

자바 메서드 호출을 관장하는 핵심 영역입니다. 물리적으로 연속된 메모리일 필요는 없습니다.

Frame (스택 프레임) 메서드가 호출될 때마다 하나의 프레임이 `push` 되고, 종료되면 `pop` 됩니다.

#### Local Variable Array

0부터 시작하는 인덱스를 가진 배열입니다.

**Slot**

기본 단위는 `Slot`입니다.
- 1 Slot: `boolean`, `byte`, `char`, `short`, `int`, `float`, `reference`, `returnAddress`
- 2 Slot: `long`, `double`

> 2 Slot은 32비트 JVM 이슈 때문에 Atomic 하지 않을 수 있습니다.
{: .prompt-warning }

**인덱싱**

인스턴스 메서드의 경우 `index 0`은 무조건 `this` 레퍼런스가 할당됩니다. 메서드 파라미터들이 순서대로 할당되고, 그 뒤에 메서드 내부 지역 변수들이 할당됩니다.

**최적화**

컴파일러(javac) 흐름 분석에 따라, 변수의 유효 범위(Scope)가 끝나면 그 슬롯은 다른 변수에 의해 재사용될 수 있습니다.

#### Operand Stack

CPU의 범용 레지스터(R1, R2 등) 역할을 대신하는 LIFO 작업 공간입니다. JVM은 Stack-based Architecture를 따르기 때문입니다. 예를 들어, `iadd` (int 더하기) 명령어가 실행되면, Operand Stack에서 상위 2개의 값을 `pop`하여 더한 뒤, 결과를 다시 `push` 합니다.

**Type Safety**

런타임이 아닌 클래스 로딩(Verification) 단계에서 데이터 타입 흐름이 검증됩니다. 예를 들어, 스택에 `int`를 넣고 `float` 연산자(`fadd`)를 수행하려 하면 검증 단계에서 실패합니다.

#### Frame Data (Dynamic Linking & Return Address)

**Dynamic Linking**

현재 프레임의 메서드가 속한 클래스의 Runtime Constant Pool에 대한 참조를 가집니다. 메서드 내 다른 메서드나 필드를 참조할 때(Symbolic Reference), 이 링크를 통해 실제 메모리 주소(Direct Reference)를 찾아냅니다.

**Exception Dispatch**

예외 발생 시 catch 블록을 찾기 위해 이 데이터를 참조합니다.

### 3. Native Method Stack (C Stack)

JNI(Java Native Interface)를 통해 호출되는 C/C++ 코드가 사용하는 스택입니다. 구현체(HotSpot 등)에 따라 JVM Stack과 Native Stack을 통합하여 관리하기도 합니다.

#### Context Switching

자바 메서드에서 Native 메서드를 호출하면, JVM Stack을 빠져나와 Native Stack을 생성하고 확장합니다. 반대로 Native 코드에서 다시 자바 메서드(콜백 등)를 호출하면 JVM Stack으로 재진입합니다.

## 🧩 Shared Data Areas (공유 메모리 영역)

JVM 구동 시 생성되며, 모든 스레드가 공유하므로 동기화와 GC의 대상입니다.

### 4. Heap

모든 클래스 인스턴스와 배열이 할당되는 런타임 데이터 영역입니다. 실제 객체의 데이터를 저장하고, 메서드 정보나 코드는 여기 있지 않습니다.

#### 실제 구현(HotSpot) 심화

- Generational Heap: Spec은 GC 알고리즘을 강제하지 않지만, 대부분의 구현체는 "약한 세대 가설"에 따라 Young과 Old 영역으로 나눕니다.
- TLAB (Thread Local Allocation Buffer): 힙은 공유 자원이므로 객체 생성 시 락(Lock)이 필요합니다(병목 현상). 이를 해결하기 위해, Eden 영역의 일부를 각 스레드별로 작게 떼어주어(TLAB), 자신만의 구역에서는 락 없이 고속으로 객체를 할당합니다. TLAB이 꽉 찰 때만 동기화 처리를 합니다.

### 5. Method Area

논리적으로는 힙의 일부이지만, GC나 압축을 하지 않을 수도 있는 특수 영역입니다.

#### 저장 데이터 (Class Metadata)

- Type Info: 클래스/인터페이스의 풀 네임, 부모 클래스, 제어자
- Field & Method Info: 이름, 타입, 시그니처, 그리고 그 Code Attribute (바이트코드 명령어, 예외 테이블 등)
- Static Variables: 클래스 변수 (JDK 8 이후에는 힙으로 이동된 구현체도 있음, 버전별 상이)

### 6. Runtime Constant Pool

Method Area 내부 클래스 별 데이터 구조입니다. 클래스 파일의 `constant_pool` 테이블이 런타임에 로드된 형태입니다.
- Numeric Constants: 리터럴 상수값
- Symbolic References: 클래스, 메서드, 필드의 이름과 타입 정보(문자열)

#### Resolution

바이트코드는 `#1`, `#2` 같은 인덱스로 참조합니다.

실행 시점에 이 심볼릭 참조(이름)를 실제 메모리 주소(Direct Reference)로 바꾸는 과정을 Resolution이라고 하며, 이 정보가 Constant Pool에 캐싱됩니다.

## Runtime Constant Pool Resolution

자바는 "동적 로딩(Dynamic Loading)"을 지원합니다. 컴파일 시점이 아닌, 코드가 실행되는 그 순간에 클래스를 로딩하고 메모리 주소를 연결합니다. 이 과정의 핵심 기술인 "Lazy Resolution(지연 해결)"의 원리를 확인해보겠습니다.

### 바이트코드 확인

```java
public class ResolutionTest {
    public void execute() {
        HelloClass obj = new HelloClass();
        int value = obj.myField; 
        obj.sayHello();      
    }

    public static void main(String[] args) {
        new ResolutionTest().execute();
    }
}

class HelloClass {
    public int myField = 100;

    public void sayHello() {
        System.out.println("Hello!");
    }
}
```

이 코드를 컴파일한 뒤 `javap -v`로 바이트코드를 확인해보면, 작성한 코드가 JVM 명령어로 어떻게 변환되었는지 볼 수 있습니다.

```text
// javap -v ResolutionTest.class

Constant pool:
  #10 = Fieldref           #7.#11         // HelloClass.myField:I
  #14 = Methodref          #7.#15         // HelloClass.sayHello:()V
  ...

public void execute();
  Code:
    ...
     9: getfield      #10                 // Field HelloClass.myField:I
    14: invokevirtual #14                 // Method HelloClass.sayHello:()V
```

여기서 주목할 점은 `#10`과 `#14`입니다. 바이트코드 상태에서는 `HelloClass`가 메모리 어디에 있는지, `myField`가 몇 번째 오프셋인지 전혀 모릅니다. 그저 심볼릭 참조(Symbolic Reference, `#10`, `#14`)라는 기호로만 남아있을 뿐입니다.

JVM이 이 코드를 처음 실행할 때, 이 심볼을 실제 메모리 주소로 바꾸는 Resolution(해결) 과정이 일어납니다.

### Resolution의 4단계

JVM이 `getfield #10` 명령어를 처음 만났을 때, 내부적으로 다음 4단계 과정을 거쳐 실제 주소를 찾아냅니다.

#### 1. Lookup (조회)

현재 실행 중인 클래스(`ResolutionTest`)의 Runtime Constant Pool에서 `#10` 항목을 조회합니다.

#### 2. Validation (검증 & 로딩)

`#10`이 가리키는 클래스(`HelloClass`)가 현재 JVM 메모리에 로딩되어 있는지 확인합니다. 만약 로딩되지 않았다면, 이 때 동적 로딩이 발생합니다.

```text
// java -verbose:class ResolutionTest

[1] 메인 메서드 시작
[2] HelloClass 사용 직전
[0.066s][info][class,load] HelloClass source: file:/C:/lws_workspace/study/java/
Hello!
[3] 종료
```

메인 메서드가 시작된 이후에야 `HelloClass`가 로딩되는 것을 확인했습니다.

#### 3. Access Control (접근 제어 확인)

`ResolutionTest`가 `HelloClass`의 `myField`에 접근할 권한(public/private 등)이 있는지 검사합니다. 권한이 없다면 이 단계에서 `IllegalAccessError`가 발생합니다.

#### 4. Replacement (직접 참조 교체)

검증이 끝나면 심볼릭 참조를 직접 참조(Direct Reference)로 바꿉니다.
- Field(`getfield`): 객체 메모리 레이아웃 내에서 해당 필드의 오프셋을 계산합니다.
- Method(`invokevirtual`): 가상 메서드 테이블(vtable)에서 해당 메서드의 인덱스를 찾아냅니다.

```text
// java -Xlog:class+resolve=debug ResolutionTest

[1] 메인 메서드 시작
[2] HelloClass 사용 직전
...
[0.063s][debug][class,resolve] HelloClass java.lang.Object (super)
[0.063s][debug][class,resolve] ResolutionTest HelloClass ResolutionTest.java:4 (explicit)
[0.063s][debug][class,resolve] ResolutionTest HelloClass ResolutionTest.java:4
...
Hello!
[3] 종료
```

JVM이 이 로그를 출력했다는 것은 다음 3가지 조건을 모두 통과했다는 뜻입니다.
1. Lookup (조회): `ResolutionTest`의 상수 풀에서 `HelloClass`라는 심볼을 찾았다.
2. Validation (검증 & 로딩): `HelloClass`가 메모리에 로딩되어 있고(바로 윗줄 로그 `HelloClass java.lang.Object (super)`가 증거), 유효한 클래스임이 확인되었다.
3. Access Control (접근 제어): `ResolutionTest`가 `HelloClass`를 사용할 권한(public 등)이 있다.

이 모든 체크가 끝난 직후, JVM은 비로소 "Resolution(해결)" 로그를 찍습니다. 즉, 이 로그가 찍히는 그 찰나의 순간에 JVM 내부에서는 심볼릭 참조(#Number)를 실제 메모리 주소(Direct Reference)로 갈아끼우는 작업(Replacement)이 수행됩니다.

```bash
jhsdb hsdb --pid <PID>
```

실행 중인 JVM의 메모리를 HSDB로 덤프 떠서 확인해 본 결과, Constant Pool의 해당 인덱스에 실제 메모리 주소가 연결된 걸 확인할 수 있습니다.

![](/assets/img/2026-01-04-00-04-57.png)

Resolution 전이었다면 단순한 이름(String)이었겠지만, Resolution 후에는 `@0x...` 라는 물리적 주소(Direct Reference)로 바뀌어 있음을 알 수 있습니다.

### Constant Pool Cache & Bytecode Rewriting

매번 명령어를 실행할 때마다 위 4단계를 반복하면 성능이 저하됩니다. 그래서 HotSpot JVM은 Bytecode Rewriting 기법을 사용합니다.

#### Constant Pool Cache (CPCache)

Resolution이 완료된 결과(실제 메모리 주소, 오프셋 등)를 저장하기 위해, 런타임 상수 풀 옆에 CPCache라는 별도의 고속 배열을 만듭니다.

#### Bytecode Rewriting

첫 Resolution이 성공하면, JVM은 바이트코드 명령어를 런타임에 몰래 수정합니다.
- `getfield` → `getfield_quick` (또는 `fast_access`)
- 피연산자 `#1` → CPCache의 인덱스

```java
// 두 번째 실행부터 바이트코드가 메모리 상에서 이렇게 변해있음
getfield_quick [CPCache_Index]
```

이제 JVM은 복잡한 이름 찾기 과정 없이, CPCache에 저장된 오프셋을 바로 가져와서 Heap 메모리에 접근합니다. 이것이 인터프리터 방식임에도 네이티브에 준하는 속도를 내는 이유입니다.

## PermGen & Metaspace

JDK 7에서 8로 넘어가면서 클래스 메타데이터 관리의 주체가 JVM에서 OS로 변경되었습니다.

### PermGen (Permanent Generation) - JDK 7까지

힙의 일부로 존재하며, Young/Old 영역과 연속된 메모리 공간을 사용했습니다. 클래스 메타데이터, Static 변수, String Constant Pool (JDK 6까지)를 저장합니다.
- 고정된 크기 (Fixed Size)
  - JVM 시작 시 `-XX:MaxPermSize`로 크기를 지정해야 했습니다. (기본값은 64MB/82MB로 매우 작음)
  - 동적 프레임워크(Spring, Hibernate)는 런타임에 바이트코드 조작(CGLib 등)을 통해 수만 개의 프록시 클래스를 생성합니다. 이 공간이 꽉 차면 GC가 돌긴 하지만, 그래도 부족하면 여지없이 `java.lang.OutOfMemoryError: PermGen space`가 발생하며 서버가 죽습니다.
- 관리의 어려움
  - 힙의 일부이므로 Full GC가 발생할 때만 청소됩니다. 즉, 클래스 언로딩(Unloading) 비용이 매우 비쌉니다.

> String Pool은 JDK 7에서 먼저 Heap으로 이사 갔습니다.
{: .prompt-info }

### Metaspace - JDK 8부터

이제 메타데이터는 JVM의 힙이 아닌 Native Memory(OS가 관리하는 메모리)에 저장됩니다.
- 동적 확장 (Auto-Scaling)
  - 크기가 고정되지 않습니다. OS가 허용하는 한 메모리를 계속 끌어다 씁니다.
  - 개발자가 사이즈 예측 실패로 인한 OOM을 겪을 일이 획기적으로 줄어듭니다.
- Chunk Based Allocation (청크 기반 할당)
  - Metaspace는 ChunkAllocator라는 별도의 할당자를 사용합니다.
  - 클래스 로더(Class Loader) 별로 메모리 덩어리(Chunk)를 할당받아 관리합니다.
  - 특정 클래스 로더가 죽으면(예: 웹 애플리케이션 재배포), 그 로더가 관리하던 청크들을 통째로 OS에 반환할 수 있습니다. 개별 클래스 단위로 청소하던 PermGen보다 훨씬 효율적이고 파편화(Fragmentation)가 적습니다.
- 데이터의 대이동
  - 클래스 메타데이터: Metaspace (Native)
  - Static 변수 (Class Variables): Heap으로 이동
    - **이제 static 변수도 GC의 직접적인 대상이 되며, Heap 메모리를 차지합니다.**

네, 앞서 본문에서 `ResolutionTest`와 `HelloClass`로 실습한 내용을 바탕으로 **Summary** 섹션만 다시 작성했습니다. `UserService` 예시를 `ResolutionTest`로 변경하여 글의 일관성을 맞췄습니다.

## Summary (전체적인 상호작용)

앞서 살펴본 `ResolutionTest` 코드가 실행될 때, JVM의 각 메모리 영역이 어떻게 상호작용하는지 정리해 보겠습니다.

1. 로딩 (Method Area & Runtime Constant Pool)
- JVM이 시작되면서 `ResolutionTest.class`가 Method Area에 로드됩니다.
- 이때 클래스의 메타데이터와 바이트코드, 그리고 Runtime Constant Pool이 생성됩니다.
- 단, `HelloClass`는 아직 로드되지 않습니다 (Lazy Loading).

2. 메서드 실행 준비 (JVM Stack)
- `main()` 메서드에서 `execute()`를 호출하면, 스레드의 JVM Stack에 `execute()`를 위한 새로운 Frame이 푸시(Push)됩니다.
- 이 Frame 내부의 Local Variable Array에는 `this` 참조와 나중에 생길 `obj` 변수를 위한 공간이 할당됩니다.

3. 객체 생성 (Heap)
- `new HelloClass()` 명령어를 만나면, 그제야 `HelloClass`를 로딩하고 Heap 영역(가능하다면 TLAB)에 실제 인스턴스를 생성합니다.
- 생성된 객체의 주소값(Reference)은 Stack의 Local Variable Array(`obj`)에 저장됩니다.

4. 명령어 실행 (PC Register)
- PC Register는 Method Area에 있는 바이트코드 명령어(`getfield`, `invokevirtual` 등)의 주소를 하나씩 가리키며 실행을 유도합니다.

5. 참조 해결 (Dynamic Linking & Resolution)
- `getfield #10` 명령어가 실행되는 순간, JVM은 Constant Pool의 `#10` 심볼릭 참조(Symbolic Reference)를 확인합니다.
- Resolution: 아직 연결되지 않았다면, 이 시점에 `HelloClass`의 실제 메모리 주소와 `myField`의 오프셋을 찾아내어 직접 참조(Direct Reference)로 변경합니다.
- Optimization: 이후 동일한 코드가 실행될 때는 Bytecode Rewriting을 통해 `getfield_quick`으로 최적화되어, Constant Pool Cache를 통해 고속으로 메모리에 접근합니다.

이러한 유기적인 상호작용 덕분에 자바는 정적인 코드 상태를 유지하면서도, 실행 시점에는 동적으로 메모리를 연결하고 최적화할 수 있습니다.

## References
- [[Naver D2] JVM Internal](https://d2.naver.com/helloworld/1230)
- [[Oracle] The Java Virtual Machine Specification - Run-Time Data Areas](https://docs.oracle.com/javase/specs/jvms/se17/html/jvms-2.html#jvms-2.5)