---
title: 컴파일러는 내 코드를 어디까지 바꿀 수 있는가
date: 2026-04-12
categories:
  - C_Language
tags:
  - C11
  - N1570
  - Abstract_Machine
  - Observable_Behavior
  - volatile
  - Side_Effect
---

임베디드 개발을 하다 보면 최적화 옵션 하나로 동작이 달라지는 경험을 하게 됩니다. 
TI CGT 기반 작업에서 동일한 코드를 `-O0`과 `-O2`로 각각 빌드했을 때 결과가 다른 경험이 있었습니다. 컴파일러의 최적화에 대해 자세히 알고 싶었습니다. 따라서 C 표준에서 이를 어떻게 정의하고 있는지, 컴파일러는 얼마나 자유롭게 코드를 바꿀 수 있는지에 관하여 공부한 글입니다.

이 글은 아래 순서로 진행합니다.
먼저 C 표준이 프로그램의 의미를 정의하는 방식인 **추상 기계(Abstract Machine)** 개념을 설명합니다. 추상 기계가 무엇인지, 그리고 실제 구현체(전처리기, 컴파일러, 링커, 라이브러리 등)가 이 모델을 그대로 따르지 않아도 되는 이유를 다룹니다.
다음으로 구현체가 반드시 지켜야 하는 **최소 요구사항(Observable Behavior)** 세 가지를 살펴봅니다. 이 경계가 곧 컴파일러가 코드를 바꿀 수 있는 범위의 한계입니다. 특히 임베디드 환경에서 **volatile**이 이 요구사항과 어떻게 연결되는지를 중점적으로 다룹니다.
이어서 최소 요구사항과 직결된 **부수 효과(Side Effect)** 개념을 정리합니다. 부수 효과가 없는 코드는 컴파일러가 제거하거나 순서를 바꿀 수 있고, 부수 효과가 있는 코드는 그렇지 않습니다.
마지막으로 **표준이 직접 제시하는 예제**를 통해 허용 범위의 경계를 확인합니다. 번역 단위 기반 최적화, `char` 정수 승격, 부동소수점 재배치 제한, 정수 오버플로와 연산 순서의 네 가지 예제를 다룹니다. 
##  1. 추상 기계 (Abstract Machine)

> The semantic descriptions in this International Standard describe the behavior of an abstract machine in which issues of optimization are irrelevant.

국제 표준의 의미론적 기술(semantic description)은, 최적화 문제가 관련 없는 **추상 기계(abstract machine)** 의 동작을 기술한다. 

>In the abstract machine, all expressions are evaluated as specified by the semantics.  

추상 기계에서는, 모든 표현식이 의미론(semantics)에 명시된 대로 평가된다.

>An actual implementation need not evaluate part of an expression if it can deduce that its value is not used and that no needed side effects are produced (including any caused by calling a function or accessing a volatile object).  

실제 구현체는, 어떤 표현식의 값이 사용되지 않고, 필요한 부수 효과—함수 호출이나 **volatile** 객체 접근에 의해 발생하는 것을 포함하여—도 생성되지 않는다고 추론할 수 있는 경우, 해당 표현식의 일부를 평가하지 않아도 된다.

C 표준이 정의하는 것은 특정 CPU나 컴파일러의 동작이 아니라, **추상 기계(abstract machine)** 의 동작입니다. 추상 기계는 최적화 문제와 무관하게 프로그램의 의미를 기술하기 위한 이상적 실행 모델입니다.

추상 기계에서는 모든 식(expression)이 의미론(semantic description)에 명시된 대로 평가됩니다. 그러나 실제 구현체는 이 모델을 그대로 따를 의무가 없습니다. **식의 결과값**이 사용되지 않고, 필요한 **부수 효과(side effect)** 도 발생하지 않는다고 추론할 수 있다면, 해당 식의 평가를 생략하거나 순서를 바꿀 수 있습니다.

이 설계가 의미하는 것은 명확합니다. 표준은 **"무엇을 계산하는가"** 를 추상 기계로 정의하고, **"어떻게 계산하는가"** 는 구현체(전처리기, 컴파일러, 링커, 라이브러리 등)에 위임한다는 것입니다. 컴파일러는 추상 기계가 정의한 결과만 보존하면, 내부적으로 어떤 변환을 수행해도 표준에 위반되지 않습니다. 따라서 구현체는 이유와 무관하게, **최소 요구사항(Observable Behavior)** 이 보존되는 한, 자율적으로 코드변환을 수행할 수 있습니다. 
## 2. 최소 요구사항 (Observable Behavior)

적합한 구현체(conforming implementation)에 대한 **최소 요구사항**은 아래 세 가지입니다. 

> Accesses to volatile objects are evaluated strictly according to the rules of the abstract machine.  
 
  **volatile** 객체에 대한 접근은 추상 기계의 규칙에 따라 **엄격하게(strictly)** 평가되어야 한다.

> At program termination, all data written into files shall be identical to the result that execution of the program according to the abstract semantics would have produced.

프로그램 종료 시점에서 파일에 기록된 모든 데이터는 추상 의미론(abstract semantics)에 따라 프로그램이 실행되었을 경우 생성되었을 결과와 **동일해야(identical)** 한다.

> The input and output dynamics of interactive devices shall take place as specified in 7.21.3.  

대화형 장치(interactive device)의 입출력 동작은 7.21.3절에 명시된 대로 이루어져야 한다.

첫 번째 요구사항은 **volatile** 객체에 대한 접근입니다. 구현체는 **volatile**로 선언된 객체의 접근을 추상 기계의 규칙에 따라 엄격히 평가해야 합니다.  이는 추후 설명할 Example 1의 *번역 단위(translation unit) 기반 최적화*와도 관련있습니다. 구현체는 번역 단위 내부에서 변수를 레지스터에만 유지하거나, 접근 순서를 바꿀 수 있습니다. 하지만, 외부 하드웨어나 다른 실행 흐름이 해당 메모리를 직접 변경하는 경우라면 이 가정이 무너집니다. 표준은 **volatile**을 다음과 같이 정의하며, 이것이 구현체의 최적화를 제한하는 직접적인 근거가 됩니다.

> An object that has volatile-qualified type may be modified in ways unknown to the implementation or have other unknown side effects.

**volatile**로 한정된(qualified) 타입을 가진 객체는, **구현체가 알 수 없는 방식으로 수정**되거나, 또는 **다른 알 수 없는 부수 효과(side effect)** 를 가질 수 있다.

이 정의는 두 가지 경우를 포함합니다.
첫 번째는 **"구현체가 알 수 없는 방식으로 수정"** 되는 경우입니다. 컴파일러는 소스 코드 분석만으로 모든 객체 변경을 추적합니다. 그러나 하드웨어가 직접 메모리에 값을 쓰거나, ISR이 비동기적으로 변수를 수정하는 경우는 컴파일러가 알 수 없습니다. **volatile**이 없다면 컴파일러는 "이 변수는 내가 마지막으로 쓴 값 그대로다"라고 가정하고 레지스터에 캐싱합니다.
두 번째는 **"다른 알 수 없는 부수 효과"** 입니다. 읽기 행위 자체가 하드웨어 상태를 바꾸는 경우입니다. UART 수신 레지스터는 읽는 순간 내부 버퍼가 비워집니다. 컴파일러 입장에서 값을 읽는 것은 단순한 값 계산이지만, 하드웨어 입장에서는 부수 효과가 있는 연산입니다.
이 두 가지 이유로 컴파일러는 **volatile** 객체에 대해 다음의 세 가지를 할 수 없습니다. 접근 결과를 레지스터에 **캐싱**하거나, 접근 횟수를 **줄이거나**, 다른 연산과 **순서를 바꾸는** 것입니다. 소스 코드에 명시된 그대로 매번 메모리에 접근해야 합니다.

나머지 두 요구사항은 I/O에 관한 것입니다. 파일에 기록된 데이터는 추상 기계가 실행했을 때의 결과와 동일해야 하며, 대화형 장치의 입출력 순서는 프롬프트가 입력 대기 전에 반드시 출력되도록 보장되어야 합니다. 이 두 요구사항은 임베디드와 직접적인 관련이 없으므로 간략히 언급만하고 넘어가겠습니다.

이 세 가지가 C 표준이 정의하는 **observable behavior**의 전부입니다. 이 세 가지 외의 모든 것은 구현체의 재량입니다.

## 3. 부수 효과 (Side Effects)

앞서 구현체가 반드시 지켜야 할 최소 요구사항을 살펴봤습니다. 그렇다면 구현체는 어떤 기준으로 코드를 바꿔도 되는지, 바꾸면 안 되는지를 판단할까요. 그 기준이 바로 **부수 효과(Side Effect)** 입니다. 구현체는 `"식의 값이 사용되지 않고, 필요한 부수 효과도 발생하지 않는다고 추론할 수 있는 경우 평가를 생략해도 된다"` 고 했습니다. 뒤집어 말하면, 부수 효과가 있는 코드는 생략하거나 순서를 바꿀 수 없습니다. 부수 효과의 유무가 컴파일러의 자유와 제약을 나누는 실질적인 기준입니다. 아래는 표준에서 말하는 부수 효과입니다.

> Accessing a volatile object, modifying an object, modifying a file, or calling a function that does any of those operations are all side effects, which are changes in the state of the execution environment.

**volatile** 객체에 접근하거나, 객체를 수정하거나, 파일을 수정하거나, 이러한 연산을 수행하는 함수를 호출하는 것은 모두 부수 효과(side effect)이며, 이는 실행 환경 상태의 변화이다.
여기서 말하는 부수 효과란 **volatile** 객체에 접근 혹은 수정하거나, 파일을 수정하거나, 이러한 연산을 수행하는 함수를 호출하는 것 모두 **부수 효과 Side Effect** 라 하며, 이는 실행 환경 상태 변화를 의미합니다. (**실행 환경 상태 변화** 는 프로그램 내부의 계산이 아닌, **외부 세계의 변화**를 의미합니다.)

아래는 부수 효과의 간단한 예입니다. 
```c
3 + 5;      // 8을 계산한다. 외부 세계는 아무것도 바뀌지 않는다.
a = 3 + 5;  // 8을 계산하고, 메모리(a)가 바뀐다.
```
첫 줄은 계산만을 수행, 두번째 줄은 계산 이후 메모리 상태의 변화까지 발생됩니다. **메모리 상태 변화**는 **외부 세계의 변화**로 볼 수 있습니다. 따라서 이를 **부수 효과**라 할 수 있습니다. 
어떠한 부수효과도 없는 경우 표현식의 평가를 생략할 수 있습니다. 
```c
int a = 3 + 5;  // a를 이후에 전혀 쓰지 않는 경우
```
만약 위의 코드를 작성 후 변수 a를 사용하지 않는 경우, 값만이 존재하기 때문에 부수 효과가 없고 이는 컴파일러에 의해 제거될 수 있습니다. 
## 4. 표준이 보여주는 허용 범위

표준은 여러 예제를 통해 추상 기계와 구현체 사이의 허용 범위 경계를 코드로 직접 보여줍니다. 아래는 표준의 예제 중 경계를 명확히 설명하기 쉬운 몇 가지 예제를 가져왔습니다. 더 많은 예제들은 표준을 참고하시길 바랍니다. 
### Example 1 번역 단위 경계 기반 최적화 구현

> An implementation might define a one-to-one correspondence between abstract and actual semantics: at every sequence point, the values of the actual objects would agree with those specified by the abstract semantics. The keyword volatile would then be redundant.

구현체는 추상 의미론과 실제 의미론 사이에 일대일 대응을 정의할 수도 있다: 모든 시퀀스 포인트에서, 실제 객체의 값이 추상 의미론이 규정하는 값과 일치하는 방식이다. 그 경우 **volatile** 키워드는 불필요한 것이 될 것이다.

> Alternatively, an implementation might perform various optimizations within each translation unit, such that the actual semantics would agree with the abstract semantics only when making function calls across translation unit boundaries. 

또는, 구현체는 각 **번역 단위(translation unit)** 내부에서 다양한 최적화를 수행하여, 실제 의미론이 번역 단위 경계를 넘는 함수 호출 시에만 추상 의미론과 일치하도록 할 수 있다. 

> In such an implementation, at the time of each function entry and function return where the calling function and the called function are in different translation units, the values of all externally linked objects and of all objects accessible via pointers therein would agree with the abstract semantics. 

그러한 구현체에서는, 호출 함수와 피호출 함수가 서로 다른 번역 단위에 있는 경우, 각 **함수 진입(function entry)** 시점과 **함수 반환(function return)** 시점에, 외부 연결(external linkage)을 가진 모든 객체와 그를 통해 포인터로 접근 가능한 모든 객체의 값이 추상 의미론과 일치해야 한다.

> Furthermore, at the time of each such function entry the values of the parameters of the called function and of all objects accessible via pointers therein would agree with the abstract semantics. 

나아가, 그러한 각 함수 진입 시점에, 피호출 함수의 **매개변수 값과 그 매개변수를 통해 포인터로 접근 가능한 모든 객체의 값이 추상 의미론**과 일치해야 한다.

> In this type of implementation, objects referred to by interrupt service routines activated by the signal function would require explicit specification of volatile storage, as well as other implementation-defined restrictions.

이러한 유형의 구현체에서는, **signal** 함수에 의해 활성화된 인터럽트 서비스 루틴이 참조하는 객체는 **volatile** 저장소로 명시적으로 선언해야 하며, 그 외 구현체 정의 제한도 따라야 한다.

여기서 말하는 **번역 단위(translation unit)** 은 컴파일러가 독립적으로 처리하는 단위를 말하며, 이는 소스 파일 하나를 의미합니다. 
`(번역 단위는 전처리 결과를 포함하지만 여기서는 깊게 다루지 않겠습니다.)`
```
foo.c  → 컴파일 → foo.o  ┐
bar.c  → 컴파일 → bar.o  ├→ 링커 → 실행 파일
baz.c  → 컴파일 → baz.o  ┘
```
 `foo.c`를 컴파일할 때 컴파일러는 `bar.c`의 내용을 모릅니다. 각 번역 단위는 독립적으로 컴파일됩니다. 따라서 구현체의 추상 의미론은 **번역 단위 내부**와 **번역 단위 경계**로 나눌 수 있습니다.  
```c
// foo.c
int x = 0;

void foo(void) {
    x = 1;
    x = 2;
    x = 3;  // 컴파일러: x = 1, x = 2는 덮어쓰이므로
            // 레지스터에서 x = 3만 유지하고
            // 메모리에는 아직 쓰지 않아도 됨
    bar();  // ← 번역 단위 경계 (bar는 bar.c에 있음)
            // 이 시점에 반드시 x = 3을 메모리에 써야 함
}
```
구현체는 **경계(함수 호출/반환)** 에서만 외부 환경을 추상 기계와 맞추면 됩니다. 따라서 경계 안에서는 변수가 레지스터에만 존재할 수도 있습니다. 

또한 구현체(라이브러리)에서 인터럽트 서비스 루틴이 참조하는 객체는 **volatile** 저장소를 명시적으로 선언해야합니다. 
```c
// ISR과 공유하는 변수 — volatile 필수
volatile int adc_flag = 0;

interrupt void adc_isr(void) {
    adc_flag = 1;       // 메모리에 직접 씀
}

void main_loop(void) {
    while (!adc_flag) { // 매번 메모리에서 읽음
        /* 대기 */
    }
}
```
만약 adc_flag가 volatile 저장소가 아니라면, **ISR은 메모리**에 접근하지만, **main loop는 레지스터**만 봅니다. 두 실행 흐름이 서로 다른 곳을 보는 문제가 발생할 수 있습니다. 
### Example 2 `char` 산술과 정수 승격

```c
char c1, c2;
/* ... */
c1 = c1 + c2;
```
 >The ''integer promotions'' require that the abstract machine promote the value of each variable to int size and then add the two ints and truncate the sum. 
 
 "정수 승격(integer promotions)"은 추상 기계가 각 변수의 값을 `int` 크기로 승격(promote)시킨 뒤, 두 `int`를 더하고, 그 합을 잘라낸다(truncate)는 것을 요구한다.
 
 >Provided the addition of two chars can be done without overflow, or with overflow wrapping silently to produce the correct result, the actual execution need only produce the same result, possibly omitting the promotions.
 
두 `char`의 덧셈이 오버플로 없이 수행되거나, 오버플로가 묵시적으로 wrap-around되어 올바른 결과를 생성하는 경우, 실제 실행은 승격 과정을 생략하더라도 동일한 결과만 생성하면 된다.

표준의 추상 기계에서는 연산을 단순화하고, 아키텍처 의존성을 줄이며, 일관된 의미를 보장하기 위해서 **정수 승격(Integer Promotions)** 을 수행합니다. 따라서 추상 기계는 아래와 같이 연산을 수행합니다. 
```
c1 (8비트) → int (32비트)로 승격 
c2 (8비트) → int (32비트)로 승격 
			↓ 
		int + int 덧셈 (32비트 연산) 
			↓ 
		결과를 char (8비트)로 잘라냄(truncate) 
			↓ 
		c1에 저장
```
하지만, 두 연산의 **오버플로**가 발생하지 않거나, 오버플로가 묵시적으로 **warp-around**되어 올바른 결과를 생성하는 경우 정수 승격 과정이 생략되어도 됩니다. 따라서, 하드웨어가 8비트 연산을 네이티브로 지원하는 경우, 컴파일러는 최적화 옵션에 따라 정수 승격을 생략할 수 있습니다. 이는 상황에 따라 구현체에 자율성을 보장함을 보입니다. 

### Example 3 부동소수점 식의 재배치 제한

> Rearrangement for floating-point expressions is often restricted because of limitations in precision as well as range. The implementation cannot generally apply the mathematical associative rules for addition or multiplication, nor the distributive rule, because of roundoff error, even in the absence of overflow and underflow. Likewise, implementations cannot generally replace decimal constants in order to rearrange expressions.

부동소수점 표현식의 재배치는, 정밀도(precision)와 범위(range)의 제한으로 인해 종종 허용되지 않는다. 구현체는, 오버플로나 언더플로가 없는 경우에도, 반올림 오차(roundoff error)로 인해 덧셈이나 곱셈에 대한 수학적 결합 법칙(associative rule)이나 분배 법칙(distributive rule)을 일반적으로 적용할 수 없다. 마찬가지로, 구현체는 표현식을 재배치하기 위해 10진수 상수를 일반적으로 대체해서는 안 된다.

> 표준에서 제시하는 예시 코드
```c
double x, y, z;
/* ... */
x = (x * y) * z;    // not equivalent to x *= y * z;
z = (x - y) + y;    // not equivalent to z = x;
z = x + x * y;      // not equivalent to z = x * (1.0 + y);
y = x / 5.0;        // not equivalent to y = x * 0.2;
```

수학에서는 실수 연산의 결합, 분배 법칙이 항상 성립하지만, 컴퓨터의 부동 소수점은 유한한 비트로 실수를 **근사**하기 때문에 각 연산마다 반올림이 발생합니다. 이 반올림 오차로 인해 순서가 바뀌면 결과도 바뀔 수 있습니다. 따라서 구현체는 오버플로나 언더플로가 없는 경우에도 **수학적 결합 법칙(associative rule)** 이나 **분배 법칙(distributive rule)** 을 일반적으로 적용할 수 없습니다. 
#### 곱셈 결합 법칙: 오버플로 경계
```c
double x = 1e308, y = 2.0, z = 0.5;

printf("(x * y) * z = %e\n", (x * y) * z);  // inf
printf("x * (y * z) = %e\n", x * (y * z));  // 1.000000e+308
```
``` 
결과
(x * y) * z = inf
x * (y * z) = 1.000000e+308
```
``` 
계산 경로
경로 1: (1e308 × 2.0) × 0.5
        = inf × 0.5          ← 1e308 × 2.0이 double 최댓값 초과 → inf
        = inf

경로 2: 1e308 × (2.0 × 0.5)
        = 1e308 × 1.0        ← 2.0 × 0.5 먼저 계산 → 1.0
        = 1e308
```

`x * y`를 먼저 계산하면 `double`의 최댓값(`1.7e308`)을 넘어 `inf`가 됩니다. 순서만 바꿨을 뿐인데 한쪽은 `inf`, 다른 쪽은 정상 값입니다. 컴파일러가 이 재배치를 수행하면 observable behavior가 바뀌므로 위반입니다.
#### 덧셈 결합 법칙: 정밀도 소멸
```c
double a = 1e16, b = -1e16, c = 1.0;

printf("(a+b)+c = %.1f\n", (a + b) + c);  // 1.0
printf("a+(b+c) = %.1f\n", a + (b + c));  // 0.0
```
``` 
결과
(a+b)+c = 1.0
a+(b+c) = 0.0
```

원인은 **ULP**(Unit in the Last Place, 부동소숫점에서 컴퓨터가 표현할 수 있는 가장 작은 값)입니다. `1e16`의 ULP는 `2.0`입니다. `c = 1.0`이 ULP보다 작기 때문에 `1e16`과 더할 때 소멸됩니다.

```
경로 1: (1e16 + (-1e16)) + 1.0
        = 0.0 + 1.0
        = 1.0  ← 정확

경로 2: 1e16 + ((-1e16) + 1.0)
        -1e16 + 1.0 : 1.0 < ULP(2.0) → 1.0 소멸
        = 1e16 + (-1e16)
        = 0.0  ← 오답
```

마찬가지로 위의 원인으로 아래의 **소거법** 또한 문제가 될 수 있습니다. 
```c
double x = 1e+16, y = 1.0;

	printf("y           = %.20f\n", y);        // 1.0
	printf("(x + y) - x = %.20f\n", (x + y) - x); // 0.0
```
#### 10진수 상수 대체: `x / 5.0` vs `x * 0.2`
```c
double x = 0.1;

printf("x/5.0 = %.20f\n", x / 5.0);
printf("x*0.2 = %.20f\n", x * 0.2);
```
```
결과
x/5.0  = 0.02000000000000000042
x*0.2  = 0.02000000000000000389
```
원인은 `0.2`가 이진 부동소수점으로 **정확히 표현되지 않는다**는 것입니다.
```
0.2 (10진수) = 0.001100110011... (2진수, 무한 반복)
```
`double`에 저장된 `0.2`는 가장 가까운 근사값입니다. 반면 `x / 5.0`은 `5.0`이 정확히 표현 가능하므로 나눗셈이 더 정밀하게 수행됩니다.

### Example 4 정수 연산 순서와 오버플로

```c
int a, b;
/* ... */
a = a + 32760 + b + 5;
```
> The expression statement behaves exactly the same as `a = (((a + 32760) + b) + 5);` due to the associativity and precedence of these operators. 

해당 표현식 문(expression statement)은 연산자의 결합성(associativity)과 우선순위(precedence)에 따라 `a = (((a + 32760) + b) + 5);`와 정확히 동일하게 동작한다.

> On a machine in which overflows produce an explicit trap and in which the range of values representable by an int is [−32768, +32767], the implementation cannot rewrite this expression as `a = ((a + b) + 32765);` since if the values for a and b were, respectively, −32754 and −15, the sum a + b would produce a trap while the original expression would not; nor can the expression be rewritten either as `a = ((a + 32765) + b);` or `a = (a + (b + 32765));` since the values for a and b might have been, respectively, 4 and −8 or −17 and 12. 

`int`의 표현 가능 범위가 `[−32768, +32767]`이고 오버플로가 명시적 트랩(explicit trap)을 발생시키는 기계에서, `a`와 `b`의 값이 각각 `−32754`와 `−15`인 경우 `a + b`는 트랩을 발생시키지만 원본 식은 그렇지 않으므로, 구현체는 이 식을 `a = ((a + b) + 32765);`로 재작성할 수 없다.
마찬가지로 `a`와 `b`의 값이 각각 `4`와 `−8` 또는 `−17`과 `12`일 수 있으므로, `a = ((a + 32765) + b);` 또는 `a = (a + (b + 32765));` 로도 재작성할 수 없다.

> However, on a machine in which overflow silently generates some value and where positive and negative overflows cancel, the above expression statement can be rewritten by the implementation in any of the above ways because the same result will occur.

그러나 오버플로가 묵시적으로 어떤 값을 생성하고, 양수/음수 오버플로가 상쇄되는 기계에서는 동일한 결과가 발생하므로 구현체는 위의 어떤 방식으로도 재작성할 수 있다.

아래 식은 원본과 수학적 동치를 이룹니다. 
```c
a = a + 32760 + b + 5; // 원본
a = (((a + 32760) + b) + 5)
a = ((a + b) + 32765)
a = ((a + 32765) + b)
a = (a + (b + 32765))
```

하지만 정수 연산의 순서에 따라 오버플로 여부가 달라지며, 의도적으로 최종 결과가 범위 안에 들어오도록 중간 단계에서 오버플로를 회피하는 순서로 코드를 작성할 수도 있습니다. 따라서 구현체가 이 순서를 임의로 바꾸게 되면 특정 입력에서 의도하지 않은 트랩이 발생할 수 있습니다. 아래는 표준에서 제시한 트랩 상황입니다. 
#### 케이스 1 : `a = −32754, b = −15`
재작성 시도: `((a + b) + 32765)`
```
[원본] (((a + 32760) + b) + 5)
	step1: -32754 + 32760 = 6 [범위 내: OK]
	step2: 6 + (-15) = -9 [범위 내: OK] 
	step3: -9 + 5 = -4 [범위 내: OK] → 정상 
	
[재작성] ((a + b) + 32765) 
	step1: -32754 + (-15) = -32769 [범위 초과: OVERFLOW → 트랩!]
```
#### 케이스 2 : `a = 4, b = −8`
재작성 시도: `((a + 32765) + b)`
```
[원본] (((a + 32760) + b) + 5) 
	step1: 4 + 32760 = 32764 [범위 내: OK] 
	step2: 32764 + (-8) = 32756 [범위 내: OK] 
	step3: 32756 + 5 = 32761 [범위 내: OK] → 정상 
	
[재작성] ((a + 32765) + b) 
	step1: 4 + 32765 = 32769 [범위 초과: OVERFLOW → 트랩!]
```

#### 케이스 3 : `a = −17, b = 12`
재작성 시도: `(a + (b + 32765))`
```
[원본] (((a + 32760) + b) + 5) 
	step1: -17 + 32760 = 32743 [범위 내: OK] 
	step2: 32743 + 12 = 32755 [범위 내: OK] 
	step3: 32755 + 5 = 32760 [범위 내: OK] → 정상 
	
[재작성] (a + (b + 32765)) 
	step1: 12 + 32765 = 32777 [범위 초과: OVERFLOW → 트랩!]
```

## 5. 정리
C 표준은 컴파일러가 코드를 어떻게 변환하는지 직접 규정하지 않습니다. 대신 추상 기계를 통해 "무엇을 계산하는가"를 정의하고, observable behavior를 통해 "무엇을 반드시 지켜야 하는가"의 경계만 제시합니다. 이 두 가지 사이의 모든 것은 구현체의 재량입니다.
이 규칙을 이해하고 나면 임베디드 코딩에서 주의해야 할 지점들이 명확해집니다. ISR과 공유하는 변수에 **volatile**을 빠뜨리면 컴파일러가 레지스터 캐싱을 통해 해당 변수를 최적화해버릴 수 있습니다. 부동소수점 연산에서는 수학적으로 동치인 식이라도 연산 순서에 따라 결과가 달라질 수 있으며, 컴파일러는 observable behavior가 보존되는 한 이를 재배치할 수 있습니다. 정수 연산에서도 연산 순서에 따라 중간 단계에서 오버플로 발생 여부가 달라질 수 있으므로, 구현체가 수학적으로 동치인 식으로 재작성하더라도 특정 입력에서 의도하지 않은 동작이 생길 수 있습니다. 이 모든 상황이 표준이 구현체에 허용한 자유의 범위 안에서 일어나는 일입니다. 표준의 경계를 알고 코딩하는 것과 모르고 코딩하는 것은 디버깅 난이도에서 큰 차이를 만들 것입니다. 긴글 읽어 주셔서 감사합니다. 

## 참고 문헌

[1] ISO/IEC 9899:201x — N1570 Committee Draft (April 12, 2011)
 N1570은 WG14 공개 초안으로, ISO/IEC 9899:2011(C11)과 기술적으로 동일하며 아래 주소에서 열람할 수 있습니다.
> [https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf)