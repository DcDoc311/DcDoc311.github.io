---
title: _c_int00 - main() 호출 이전 C 환경 구축
date: 2026-05-12
categories:
  - Embedded
tags:
  - TI
  - CGT
  - C-Runtime
  - startup
  - _c_int00
  - EABI
  - freestanding
---
## 1. 서문

`main()` 함수가 호출 되기 이전, 이미 전역 변수는 초기값을 가지고 있고, 스택은 사용 가능한 상태이며, `static` 키워드가 붙은 함수 내 지역 변수도 표준이 요구하는 초기값으로 설정되어 있습니다. 이 글은 `main()` 함수가 호출되기 전 초기화 과정에 대해 소개합니다. CPU 리셋과 `main()` 진입 사이에서 `누가, 무엇을, 어떤 순서로, 왜` 대해 이야기합니다. 

선행 글 [TI Linker Command File Primer]({{< ref "/docs/Embedded/TI-Linker-Command-File-Primer" >}})에서 `.bss`, `.cinit`, `.stack` 섹션을 다뤘습니다. 그 섹션들은 링커가 메모리에 자리만 잡아둔 상태이며, 실제로 그 자리를 *채우는 행위*는 별도의 코드가 수행합니다. 그 코드가 본 글의 주제인 **`_c_int00`** 입니다. 본 글은 `_c_int00`이 *왜* 그 일을 해야 하고 *무엇을* 하는지에 대해 다룹니다.

본 글은 세 계층으로 구성되어있습니다. 가장 처음은 **C 표준(C11, N1570)** , 그 다음 **ABI(Application Binary Interface)**, 그리고 마지막으로 **TI CGT(Code Generation Tools)의 RTS(Run-Time Support)** 구현이 있습니다. C 표준은 결과를 강제하지만 그 방법에 대해 말하지 않고, ABI는 그 방법을 명세하며, RTS는 그 명세를 실제 코드로 구현합니다. 이 세 계층 구조를 시작에서 끝까지 이야기 하는 것이 본 글의 목적입니다.

---

## 2. C 표준이 말하는 것

### 2.1 두 개의 실행 환경

C11 표준 N1570 §5.1.2의 첫 문단은 다음과 같이 시작합니다.

> Two execution environments are defined: freestanding and hosted.
> In both cases, program startup occurs when a designated C function is called by the execution environment.
> All objects with static storage duration shall be initialized (set to their initial values) before program startup.
> The manner and timing of such initialization are otherwise unspecified.
> Program termination returns control to the execution environment.

처음과 둘째 문장은 실행 환경의 분류와 program startup에 대해 제시합니다. Freestanding은 우리가 흔히 말하는 Standalone 프로그램의 OS가 없는 임베디드 환경을 말하며, Hosted는 OS가 있는 환경을 말합니다. 표준이 사용하는 표현은 "a designated C function is called by the execution environment" 입니다. 표준은 지정된 함수가 *호출된다*는 사실만 말하고, 그 함수가 *무엇인지*  현 문단에서 설명하지 않습니다.

세 번째 문장은 표준이 부과하는 **유일한 의무**입니다. `shall be initialized` — 정적 저장 기간 객체는 program startup 이전에 초기값을 가지고 있어야 합니다. 임베디드 시스템에서 `.bss` 제로화나 `.cinit` 처리가 굳이 필요한 C표준상의 근거가 이 한 문장입니다. 이 근거(의무)가 없다면 시스템은 부트 시 RAM을 건드릴 C표준상의 이유가 사라지게 됩니다.

네 번째 문장이 본 글의 주제입니다. `The manner and timing ... are otherwise unspecified`. 표준은 *결과*(초기화된 상태)를 강제하지만, *어떻게* 그 상태에 도달할지와 *언제* 도달할지는 대해서는 강제하지 않습니다. 이 문장이 ABI와 툴체인 구현에 대한 자유를 제공합니다.

다섯 번째 문장은 종료 동작도 실행 환경에 위임됨을 말합니다. 임베디드에서 `main()` 반환 이후의 처리에 대한 표준상의 근거(*사실상은 근거의 부재입니다. 실행 환경에 위임했기 때문입니다.* ) 가 여기에 있습니다.

### 2.2 Hosted와 Freestanding의 분기

C11 표준 N1570 §5.1.2.2.1은 Hosted 환경의 startup 함수를 명시적으로 고정합니다. 

> The function called at program startup is named `main`. The implementation declares no prototype for this function. It shall be defined with a return type of `int` and with no parameters: `int main(void) { /* ... */ }` or with two parameters [...]; or in some other implementation-defined manner.

Hosted에서는 표준이 함수 *이름*까지 결정합니다. `main`이라는 이름은 우연이 아니라 표준의 결정이며, 반환 타입과 파라미터도 두 가지 형식이 허용됩니다. 마지막 부분의 "in some other implementation-defined manner"라는 예외 조항은 흥미롭지만, 본 글에서는 깊이 들어가지 않겠습니다. 

Hosted와 대비되는 §5.1.2.1이 본 글의 핵심입니다.

> In a freestanding environment (in which C program execution may take place without any benefit of an operating system), the name and type of the function called at program startup are implementation-defined.
> Any library facilities available to a freestanding program, other than the minimal set required by clause 4, are implementation-defined.
> The effect of program termination in a freestanding environment is implementation-defined.

세 문장 모두 `implementation-defined`로 끝납니다. 표준은 freestanding 환경에서 startup 함수의 이름, 타입, 라이브러리 범위, 종료 효과 — 모두 구현체에 위임하고 있습니다. 

`_c_int00`이라는 함수명이 C표준 아래에서 등장할 수 있는 근거가 위의 첫 문장입니다. 이름뿐만 아니라 *타입*까지 위임된다는 점에 주목할 점입니다. `_c_int00`이 인자 없는 `void` 반환 함수인 것과 ARM Cortex-M의 `Reset_Handler`가 인터럽트 벡터 진입점인 것 모두 이 한 문장에 의해 정당화 됩니다. 

"implementation-defined"는 단순히 "구현체 자유"를 뜻하는 것 만은 아닙니다. C 표준은 §4에서 "An implementation shall be accompanied by a document that defines all implementation-defined ... characteristics"라고 명시합니다. 즉, implementation-defined로 구현된 항목은 *반드시 문서화되어야 한다*.는 것입니다. 이것이 다음 절에서 ABI 문서를 참조하게 되는 근거가 됩니다.

### 2.3 Freestanding 라이브러리의 최소 집합

표준이 freestanding 환경에 *보장하는* 라이브러리는 단 9개 헤더뿐입니다. 
(§4):
`<float.h>`, `<iso646.h>`, `<limits.h>`, `<stdalign.h>`, `<stdarg.h>`, `<stdbool.h>`, `<stddef.h>`, `<stdint.h>`, `<stdnoreturn.h>`

목록에 없는 것이 더욱 흥미롭습니다. `<stdio.h>`, `<stdlib.h>`, `<string.h>`, `<math.h>` — 임베디드 개발자가 자주 사용하는 헤더들은 freestanding에서는 표준이 보장하지 않습니다. TI CGT, GCC newlib, IAR DLib 등이 이 헤더들을 제공하는 것은 표준상의 의무가 아닌, 각 툴체인의 자체적인 확장입니다.

> `<stdalign.h>`와 `<stdnoreturn.h>`는 C11에서 추가된 헤더입니다. 본 글은 N1570(C11 Committee Draft) 기준이며, C99 freestanding 구현체에서는 위 목록 중 7개만 보장됩니다.

### 2.4 정리: C표준이 보장하는 것과 아닌것

위의 내용을 정리하면 아래의 표와 같습니다. 

| 항목                      | 표준이 보장하는 것                  | 표준이 보장하지 않는 것 |
| ----------------------- | --------------------------- | ------------- |
| 정적 저장 기간 객체             | program startup 시점에 초기값을 가짐 | 초기화의 방식과 시점   |
| Freestanding startup 함수 | 호출된다는 사실                    | 이름, 타입        |
| Freestanding 라이브러리      | 9개 헤더                       | 그 외 모든 것      |
| Freestanding 종료         | 환경으로 제어 반환                  | 반환 효과         |

표준에서 보장하지 않는 것이 다음 두 계층 (ABI와 RTS 구현)에서 채워야 하는 공간입니다.

---

## 3. ABI (Application Binary Interface)

### 3.1 ABI의 역할

ABI(Application Binary Interface)는 표준이 implementation-defined로 정의한 항목들을 *바이너리 수준 정의*으로 명세하는 문서입니다. C6000 아키텍처의 경우 **SPRAB89B (TMS320C6000 Embedded Application Binary Interface)** 가 이 역할을 합니다.

ABI가 다루는 항목은 C표준에서 보장하지 않는 영역과 정확히 대응합니다.

- **진입점의 이름과 호출 방식**: 표준이 보장하지 않은 startup 함수의 이름을 명시함.
- **레지스터 사용 규약**: 어떤 레지스터가 스택 포인터인지, 데이터 페이지 포인터인지, callee-saved인지를 명시함.
- **함수 호출 규약**: 인자가 어떤 레지스터를 통해 전달되는지, 반환값이 어디에 놓이는지 명시함.
- **섹션 배치 규약**: 어떤 섹션이 어떤 의미를 가지는지, 어떻게 배치되어야 하는지 명시함.

C표준의 ¶4("manner and timing ... otherwise unspecified")가 무엇으로 채워지는지에 대해 ABI가 정의합니다.

### 3.2 C6000 EABI의 핵심 항목

SPRAB89B §3.2가 명시하는 레지스터 역할 중 두가지 사항이 본 글과 직접적으로 연관있습니다. 

1. **B15는 스택 포인터(SP)** 로 사용됩니다. callee-saved(Child) 레지스터이며, 시스템 전체가 B15를 SP로 가정합니다.
2. **B14는 데이터 페이지 포인터(DP)** 로 사용됩니다. 마찬가지로 callee-saved이며, DP-relative 섹션(`.neardata`, `.bss`, `.rodata`)에 대한 near 주소 지정의 기준점입니다.

다음은 SPRAB89B §3.3의 함수 호출 시 인자 전달 항목입니다. 
이 항목은 main에 국한된 항목이 아니라 모든 함수 호출에 적용되는 일반 항목입니다. 처음 10개 인자는 선언된 순서대로 A4, B4, A6, B6, A8, B8, A10, B10, A12, B12 순서로 레지스터에 배정되며, 32~64비트 인자는 짝수 레지스터(하위)와 홀수 레지스터(상위)의 쌍으로, 나머지 인자는 SP+4부터 스택에 배치됩니다. 
이 항목이 본 글에서 의미를 갖는 지점은 그 순서의 첫 두 자리(`A4`, `B4`)입니다. 
본 글의 §4.4에서 보시겠지만, args_main이 main을 호출할 때 argc가 첫 자리(A4), argv가 둘째 자리(B4)에 놓이는 것은 main을 위한 특별 규칙이 아니라, 모든 함수에 적용되는 이 일반 순서에서 그대로 따라 나온 결과입니다. 

### 3.3 ABI의 한계와 RTS의 자리

ABI는 *어떤 함수가 진입점이고, 레지스터의 역할이 무엇이며, 어떻게 호출하는가*를 정의합니다. 그러나 ABI 역시 모든 것을 정의하진 않습니다.

ABI가 정의하지 않은 것: 진입점에 진입한 직후 *구체적으로 어떤 코드가 실행되어야 하는가*. SP를 어떤 심볼로부터 어떻게 로드할지, `.cinit` 테이블의 정확한 인코딩 포맷, 사용자가 C 런타임 초기화 이전에 끼어들 수 있는 hook의 존재 여부 등, 이런 항목들은 ABI가 직접 명세하지 않거나, 명세하더라도 *수준이 비교적 추상적*입니다.

이러한 명세에 부족함을 채워주는 것이 **TI CGT의 RTS 구현**입니다. RTS는 ABI 명세를 만족하는 구체적 코드 (`boot.c`, `args_main.c`, `autoinit.c`, `cpy_tbl.c`) 등을 제공합니다.

> **타 아키텍처 비교 (참고):** ARM Cortex-M의 경우 AAPCS(ARM IHI 0042D)가 ABI에 해당하며, CMSIS-Core가 RTS에 해당한다고 합니다. C6000에서 SP 설정이 `_c_int00` 내부의 *소프트웨어*로 이루어지는 것과 달리, ARM Cortex-M에서는 *하드웨어*가 벡터 테이블의 첫 워드를 MSP에 자동 로드한다고 합니다. 즉 ABI가 채우는 정의 영역 자체가 아키텍처마다 다릅니다.

---

## 4. TI CGT의 구현: `_c_int00`

이 절이 본 글의 핵심입니다. 표준이 정의하지 않고 ABI가 명세한 영역을, TI CGT의 RTS가 실제로 어떻게 채우는지를 다룹니다.

본 글에서 인용하는 코드는 CGT (`<CGT_install>/lib/src/`)의 `boot.c`와 `args_main.c`에서 추출했습니다. **CGT v7.4.x** 기준이며, RTS 소스의 세부는 CGT 버전에 따라 달라질 수 있습니다.

### 4.1 `_c_int00`의 위치와 선언

SPRUI03 §3.3.1은 `_c_int00`을 다음과 같이 정의합니다.

> The function `_c_int00` is the startup routine (also called the boot routine) for C/C++ programs. It performs all the steps necessary for a C/C++ program to initialize itself. The name `_c_int00` means that it is the interrupt handler for interrupt number 0, RESET, and that it sets up the C environment.

이름에 "int00"이 들어가는 이유가 첫 문단에서 설명됩니다. C6000 아키텍처에서 인터럽트 0번은 RESET 인터럽트이며, `_c_int00`은 그 인터럽트의 핸들러로도 동작할 수 있도록 설계되었습니다. 

실제 코드에서 `_c_int00`은 일반 C 함수가 아닌 `__interrupt` 키워드가 붙은 함수로 선언됩니다.

```c
extern void __interrupt c_int00()
{
    /* ... */
}
```

`__interrupt`는 TI CGT의 확장 키워드로, 컴파일러에게 일반적인 함수 prologue/epilogue를 생성하지 않도록 지시합니다. 리셋 직후에는 callee-saved 레지스터를 저장할 의미 있는 위치도 없고, 리턴 시퀀스도 적절하지 않기 때문입니다. (복귀할 필요가 없음)

링커는 ELF 헤더의 진입점 필드(`e_entry`)에 `_c_int00`의 주소를 기록합니다. CPU는 리셋 후 이 주소로 점프하며, 그 시점에서 위의 코드가 실행되게 됩니다.

### 4.2 스택 포인터와 데이터 페이지 포인터 초기화

리셋 직후 CPU 레지스터의 값은 정의되지 않은 상태입니다. SP(B15)에 유효한 주소가 없으면 어떤 함수 호출도 불가능하므로, `_c_int00`이 가장 먼저 수행하는 일은 SP 설정입니다.

`boot.c`에서 SP/DP 초기화 부분은 다음과 같습니다. 

```c
/* SP 초기화 — B15 */
#ifndef __TI_EABI__    /* COFF ABI */
   __asm("\t   MVKL\t\t   __STACK_END - 4, SP");
   __asm("\t   MVKH\t\t   __STACK_END - 4, SP");
#else                  /* EABI */
   __asm("\t   MVKL\t\t   __TI_STACK_END - 4, SP");
   __asm("\t   MVKH\t\t   __TI_STACK_END - 4, SP");
#endif

   __asm("\t   AND\t\t   ~7,SP,SP");   /* 8바이트 정렬 — COFF·EABI 공통 */

/* DP 초기화 — B14 */
#ifndef __TI_EABI__    /* COFF ABI */
   __asm("\t   MVKL\t\t   $bss,DP");
   __asm("\t   MVKH\t\t   $bss,DP");
#else                  /* EABI */
   __asm("\t   MVKL\t\t   __TI_STATIC_BASE,DP");
   __asm("\t   MVKH\t\t   __TI_STATIC_BASE,DP");
#endif
```

`__TI_STACK_END`는 **링커가 정의하는 심볼**입니다. 링커 스크립트에서 `.stack` 섹션이 배치되면 링커가 자동으로 이 심볼을 그 끝 주소로 설정합니다. 선행 글 [TI Linker Command File Primer]({{< ref "/docs/Embedded/TI-Linker-Command-File-Primer" >}})에서 다룬 `.stack` 섹션의 정의가 여기에서 사용됩니다.

`-4` 빼기는 ABI 규약입니다. C6000에서 SP는 스택의 끝 주소가 아니라 *끝에서 한 워드 안쪽*을 가리키도록 합니다.

`AND ~7,SP,SP`는 SP를 8바이트 경계로 정렬하는 명령입니다. v7.4.x에서는 8바이트 정렬이지만, 후기 CGT의 C66x 빌드에서는 16바이트 정렬이 요구된다고 합니다. 정렬 단위는 CGT 버전과 디바이스 계열에 따라 다르므로 주의할 필요가 있습니다. 

`MVKL`/`MVKH`는 32비트 즉시값 로드의 표준 패턴입니다. C6000은 단일 명령으로 16비트 즉시값만 로드할 수 있으므로, 32비트 주소를 로드하려면 하위 16비트와 상위 16비트를 각각 두 번에 나누어 로드해야 합니다.

DP 초기화에서는 COFF 심볼과 EABI 심볼의 차이점이 드러납니다. 같은 `boot.c`의 COFF 분기와 EABI 분기를 비교하면 다음과 같습니다. 

| 위치     | COFF 심볼       | EABI 심볼            | 의미                  |
| ------ | ------------- | ------------------ | ------------------- |
| SP 끝   | `__STACK_END` | `__TI_STACK_END`   | 스택의 가장 높은 주소        |
| DP 베이스 | `$bss`        | `__TI_STATIC_BASE` | DP-relative 영역의 기준점 |

COFF에서 DP는 `$bss` 즉 `.bss` 섹션만을 가리키는 기준점이었습니다. EABI에서는 `__TI_STATIC_BASE`로 변경되었으며, 이 심볼은 `.bss` 하나만이 아니라 **`.neardata` + `.rodata` + `.bss`로 구성된 "near group"의 기준점**입니다. 단순한 심볼 이름 변경이 아니라 메모리 모델 자체의 일반화가 추가되었습니다. EABI 링커 스크립트에서 다음과 같은 패턴이 등장하는 이유가 여기에 있습니다. 

```ruby
GROUP (NEAR_DP_RELATIVE) {
    .neardata
    .rodata
    .bss
} > BMEM
```

이 그룹의 중심을 가리키는 단일 심볼(`__TI_STATIC_BASE`)을 DP에 로드함으로써, 세 종류의 정적 데이터를 모두 동일한 DP-relative 주소 지정 방식으로 다룰 수 있게 됩니다. 

### 4.3 변수 초기화: `_system_pre_init()`과 `.cinit` 테이블

SP와 DP가 설정되면, 다음으로 정적 저장 기간 객체들이 표준 §5.1.2가 요구하는 초기값을 가지도록 값을 초기화합니다. 

`boot.c`에서 이 단계는 놀랍게도 단 한 줄입니다. 

```c
if(_system_pre_init() != 0)  AUTO_INIT();

_args_main();
```

#### 4.3.1 `_system_pre_init()` 사용자 hook

`_system_pre_init`은 RTS가 **weak symbol**로 제공하는 함수입니다. RTS의 기본 구현은 단순합니다.

```c
int _system_pre_init(void)
{
    return 1;
}
```

사용자가 자신의 코드에서 동일한 이름의 함수를 정의하면, 링커는 weak symbol(RTS의 기본 구현)을 strong symbol(사용자 정의)로 덮어씁니다. 반환값이 0이 아니면 `AUTO_INIT()`이 실행되고, 0이면 건너뜁니다.

이 hook의 용도는 명확합니다. C 런타임 초기화 *이전에* 수행되어야 하는 작업들 (PLL/클럭 트리 설정, DDR 컨트롤러 초기화, 캐시/MMU 사전 설정) 이 있을 때 사용자가 끼어들 수 있게 합니다. 가령 `.bss` 영역이 외부 메모리에 배치되어 있다면, DDR 컨트롤러를 활성화하기 전에는 zero-initialization 자체가 불가능합니다. 이 경우 `_system_pre_init` 내에서 DDR을 먼저 초기화해야 합니다. C표준이 정의하지 않고고 ABI도 명세하지 않은 영역을, 구현체가 자체적인 확장을 수행한 사례입니다. 

#### 4.3.2 `AUTO_INIT()` — EABI와 COFF의 분기

`AUTO_INIT()`은 `<CGT_install>/lib/src/autoinit.h`에 정의된 매크로이며, EABI와 COFF ABI에서 분기합니다. 

```c
#ifdef __TI_EABI__
  #define AUTO_INIT()  _auto_init_elf()
#else
  #define AUTO_INIT()  _auto_init(cinit)
#endif
```

이 분기 뒤에 있는 두 함수의 차이가 EABI와 COFF ABI의 핵심 차이이며, 이어지는 절(§4.3.3~§4.3.5)에서 구체적으로 다룹니다. SPRU514Y의 다음 문장이 차이를 말해줍니다.

> The name `.cinit` is used primarily to simplify migration from COFF to ELF format and the `.cinit` section created by the linker has nothing in common (except the name) with the COFF cinit records.

같은 `.cinit`이라는 이름을 쓰지만, COFF와 EABI에서 그 내부는 *공통점이 없습니다*. 이름은 마이그레이션 편의를 위해 유지되었을 뿐입니다.

#### 4.3.3 COFF `.cinit` 레코드 포맷

COFF ABI에서 `.cinit` 섹션은 변수별 레코드의 단순한 나열입니다. (SPNU151V Figure 6-5).

```
COFF .cinit 레코드 (변수당 1개):
┌─────────────┬─────────────┬───────────────────────┐
│ size (4B)   │ address (4B)│ data (variable bytes) │
└─────────────┴─────────────┴───────────────────────┘
```

각 레코드는 *어디에*(address), *얼마나*(size), *무엇을*(data) 복사할지 정의합니다. 압축은 없으며 각 변수가 별도의 레코드를 차지합니다. `_auto_init`은 이 테이블을 처음부터 끝까지 순회하며 각 레코드의 data를 address가 가리키는 RAM 위치로 복사합니다.

#### 4.3.4 EABI `.cinit` 테이블 포맷

EABI는 구조를 두 단계로 분리합니다. (SPRUI04 §8.9.2.4).

**1단계 자동 초기화 테이블 (출력 섹션 단위):**

```
__TI_CINIT_Base ─┐
                 ▼
┌──────────────────────────────────────┐
│ Record 0: load_addr │ run_addr       │  ← 출력 섹션 1
├──────────────────────────────────────┤
│ Record 1: load_addr │ run_addr       │  ← 출력 섹션 2
└──────────────────────────────────────┘
                 ▲
__TI_CINIT_Limit ┘
```

**2단계 인코딩된 초기화 데이터 (`load_addr`가 가리키는 곳):**

```
┌─────────────┬──────────────────┐
│ handler idx │ encoded data ... │
│   (8 bits)  │   (variable)     │
└─────────────┴──────────────────┘
```

1단계의 각 레코드는 단순히 "어느 위치에서(`load_addr`) 데이터를 읽어 어느 위치로(`run_addr`) 복사할지"의 한 쌍입니다. 복사와 디코딩은 2단계 데이터 첫 8비트 handler index가 결정합니다. `__TI_Handler_Table_Base`에 등록된 함수 포인터 배열에서 해당 인덱스의 핸들러를 가져와 디코딩과 복사를 위임합니다. 핸들러는 압축 방식(LZSS, RLE)이나 zero-initialization 같은 처리 방식별로 RTS 내에 분산되어 있습니다. (`copy_decompress_lzss.c`, `copy_decompress_rle.c`, `copy_zero_init.c` 등). 쉽게 이야기해, load_addr에 핸들러 인덱스를 보고 디코딩 함수를 결정하며, 디코딩 함수를 통해 encoded data를 디코딩하고 그 데이터를 run_addr에 옮깁니다. 핸들러의 내부 구현 자체는 본 글의 범위를 넘어가며, 후속 글에서 상세히 다루겠습니다.

#### 4.3.5 COFF vs EABI 비교

위 내용을 표로 정리하면 다음과 같습니다.

| 항목              | COFF ABI                         | EABI                                                      |
| --------------- | -------------------------------- | --------------------------------------------------------- |
| 레코드 단위          | 변수당 1개                           | 출력 섹션당 1개                                                 |
| 레코드 구조          | (size, address, data...)         | (load_addr, run_addr) → 인코딩된 데이터                          |
| 압축              | 없음                               | LZSS, RLE, zero_init 핸들러 선택 가능                            |
| `.bss` 제로화      | 별도 처리 또는 cinit이 0 데이터 명시 복사      | cinit 핸들러 시스템의 `zero_init`로 통합                            |
| `--ram_model` 시 | 링커가 `cinit = -1` 설정 → 부트 시 처리 없음 | 링커가 `__TI_CINIT_Base == __TI_CINIT_Limit` 설정 → 부트 시 처리 없음 |
| `.data` 섹션 상태   | initialized                      | 컴파일러는 initialized로 생성, 링커가 uninitialized로 변경              |

EABI에서 `.data` (엄밀히는 `.neardata`/`.fardata`) 섹션은 컴파일러가 처음 생성할 때는 initialized 상태지만, 링커가 이 섹션의 초기값을 `.cinit` 테이블로 흡수한 후 `.data` 자체는 uninitialized 섹션으로 변경합니다. 이 변경으로 ELF 출력 파일의 `.data` 섹션이 실제 데이터를 포함하지 않게 되어 파일 크기가 줄어듭니다.

### 4.4 `main()` 진입: 3계층의 협력

`AUTO_INIT()`이 끝나면 `_args_main()`이 호출됩니다. `<CGT_install>/lib/src/args_main.c`의 코드는 아래와 같습니다.

```c
int _args_main(void)
{
    register int argc = 0;
    register char **argv = 0;

    if (_symval(&__c_args__) != NO_C_ARGS)
    {
        register ARGS *pargs = (ARGS*)_symval(&__c_args__);
        argc = pargs->argc;
        argv = pargs->argv;
    }

    return main(argc, argv);
}
```

이 함수는 매우 단순합니다. `__c_args__`는 링커가 정의하는 심볼이며, 사용자가 `--args=<size>` 링커 옵션을 사용하지 않으면 링커는 이 심볼에 `NO_C_ARGS` (`-1`)를 기록합니다. 임베디드의 일반적인 경우 `--args`는 사용되지 않으므로 `argc=0, argv=NULL`로 `main` 함수가 호출됩니다.

주목할 만한 부분은 마지막 코드입니다.

```c
return main(argc, argv);
```

`_args_main`은 항상 두 인자를 전달하며 `main`을 호출합니다. 그러나 임베디드 코드에서 거의 모든 `main`은 다음 형태로 정의됩니다.

```c
int main(void)
{
    /* ... */
}
```

호출자(`_args_main`)는 두 인자를 전달하는데, 피호출자(`main`)는 인자를 받지 않습니다. 그런데 이 둘은 서로 다른 번역 단위에 있습니다. ( `args_main.c`와 사용자의 `main.c`는 따로 컴파일됨) 컴파일러가 둘을 한 번에 비교하는 일이 없으므로, 인자 개수 불일치는 컴파일 단계에서 드러나지는 않습니다. (이는 main이 특별해서가 아니라 모든 함수에 해당함) 만약 이것이 일반 함수였다면, 이런 번역 단위 간 시그니처 불일치는 컴파일 에러가 아니라 런타임 미정의 동작(undefined behavior)으로 이어집니다. 그런데 main에서는 이 호출이 미정의 동작조차 되지 않고 정상 동작하게 됩니다.
#### 4.4.1 Layer 1 — C 표준의 main 정의

§5.1.2.2.1의 아래 두 문장이 이 변칙을 허용합니다. 

> It shall be defined with a return type of `int` and with no parameters: `int main(void) { /* ... */ }` or with two parameters [...] `int main(int argc, char *argv[]) { /* ... */ }` or equivalent; or in some other implementation-defined manner.
>
> The implementation declares no prototype for this function.

C 표준이 `main`에 부여한 *두 가지 예외* 입니다. 

첫째, C 표준에서 `int main(void)`와 `int main(int, char**)`를 **둘 다 문제가 없습니다.** 사용자가 어느 쪽을 선택해도 표준에 문제되지 않습니다.

둘째 **"The implementation declares no prototype for this function."** 보통 C 함수는 호출 이전에 prototype 선언이 필요하고, 호출과 정의의 시그니처가 일치해야 합니다. 하지만 `main`만은 예외입니다. 표준은 `main`의 prototype을 명시적으로 *선언하지 않는다* 정의합니다.

이 두 문장이 결합되면, `_args_main`이 `main`을 어떤 인자로 호출하든 그 호출은 표준에 위배되지 않게 됩니다.

#### 4.4.2 Layer 2 — ABI 파라미터 전달 정의

C6000 EABI에서 함수의 처음 두 인자가 다음 레지스터로 전달됨을 정의하고 있습니다.

| 인자 위치          | 레지스터 | 이 경우의 의미 |
| -------------- | ---- | -------- |
| 첫 번째 (int)     | A4   | `argc`   |
| 두 번째 (pointer) | B4   | `argv`   |

핵심은 **호출자(`_args_main`)는 두 레지스터(A4, B4)를 *무조건* 세팅한다**는 사실입니다. 호출자는 피호출자(`main`)의 시그니처를 *알 필요가 없습니다*. ABI에서 인자 전달의 위치를 *호출자 측에 결정적으로* 정의해 두었기 때문입니다.

#### 4.4.3 Layer 3 — TI 구현체

이제 사용자 `main`의 두 시그니처가 각각 어떻게 처리되는지를 보겠습니다.

**Case A: `int main(void)`로 정의된 경우.**
함수 본문이 A4와 B4를 *읽지 않습니다*. 두 레지스터에 어떤 값이 들어 있든 사용되지 않고 함수 내에서 다른 용도로 재사용됩니다. 호출자가 세팅한 값은 단순히 사용되지 않을 뿐, 어떤 부작용도 없습니다.

**Case B: `int main(int argc, char **argv)`로 정의된 경우.**
함수 본문이 A4와 B4를 읽습니다. `--args` 옵션을 사용했다면 실제 인자 값을, 그렇지 않다면 `(0, NULL)`을 받습니다.

링커 차원에서도 두 정의는 같은 심볼(`_main`)을 생성합니다. C는 C++와 달리 name mangling을 하지 않으므로, 시그니처가 다르더라도 심볼 이름은 동일하게 됩니다. 링커는 *이름만* 매칭하며, 시그니처를 검사하지 않습니다.

#### 4.4.4 main(argc, argv)와 Layer의 관계

이 한 줄을 다시 확인하겠습니다. 

```c
return main(argc, argv);
```

- **Layer 1 (C 표준)** `main`에 prototype 예외와 두 시그니처 허용이라는 *예외*를 정의하지 않았다면, 이 호출은 표준 위반입니다.
- **Layer 2 (ABI)** 호출자 측에서 A4=argc, B4=argv를 결정적으로 정해 두지 않았다면, 호출자는 피호출자의 시그니처를 알아야 합니다.

두 계층이 한 결과로 합쳐져 `main` 호출이 오류 없이 동작하게 됩니다. 어느 한 계층이라도 다르게 설계되었다면 동작하지 않거나, 사용자가 `main`을 정의할 때마다 `--args` 옵션을 의식해야 했을 것입니다.

### 4.5 `main()` 반환 이후

`_args_main`은 `main`의 반환값을 그대로 자신의 반환값으로 사용합니다. `boot.c`에서 `_args_main()` 호출 이후의 처리는 두 가지 패턴이 일반적으로 알려져 있습니다.

```c
/* 패턴 A */
exit(_args_main());

/* 패턴 B */
_args_main();
abort();   /* 또는 while(1); */
```

어느 패턴이 사용되는지는 CGT 버전과 빌드 설정에 따라 다르며, 정확한 코드는 `boot.c`의 끝부분에서 확인할 수 있습니다. 

본 글의 맥락에서 중요한 것은 *어떤 패턴인가*가 아니라, **freestanding 환경에서 `main()` 반환의 정의가 C 표준상 implementation-defined임** 이라는 것입니다. C 표준이 "환경으로 제어가 반환된다"는 사실 외에는 정의하지 않았기 때문에, 임베디드 시스템마다 ( 같은 TI CGT 안에서도) 종료 처리가 다를 수 있는 것입니다.

대부분의 임베디드 코드에서 `main()`은 무한 루프로 끝나며 반환하지 않습니다. `main()`이 반환되는 것은 시스템 설계상의 예외적 상황으로 취급되며, 그 이후의 처리는 *복구할 수 없는 상태*에 진입한다는 신호로 해석되는 경우가 많습니다.

---
## 5. 마무리

세 계층을 다시 정리합니다. 

**Layer 1 — C 표준 (N1570).** 정적 저장 기간 객체가 program startup 시점에 초기값을 가져야 한다고 정의 합니다. 그러나 *방법과 시점*에 대해서는 이야기 하지 않습니다. Freestanding 환경에서는 startup 함수의 이름과 타입까지 implementation-defined로 위임하고 있습니다. 

**Layer 2 — ABI (SPRAB89B).** 표준의 미정의를 *바이너리 수준의 명세*로 정의합니다. 진입점 이름, 레지스터 역할(B15=SP, B14=DP), 함수 호출 규약(A4=arg1, B4=arg2), 섹션 배치 등, C 표준이 implementation-defined로 위임한 항목들을 명시적 문서로 정의합니다.

**Layer 3 — TI CGT RTS.** ABI 명세를 실제 코드로 구현합니다. `_c_int00`은 ABI가 정한 진입점 이름의 구체적 구현이며, SP/DP 초기화 어셈블리는 ABI가 정한 레지스터 역할의 구현 코드입니다. `.cinit` 테이블의 포맷, `_system_pre_init` hook, `_args_main`의 인자 전달, 모두 ABI가 직접 명세하지 않은 정의에서 구현체가 결정한 항목들입니다.

세 계층 중 어느 하나도 다른 계층을 *대체할 수* 없습니다. 표준만으로는 코드 한 줄도 컴파일되지 않고, ABI만으로는 실제로 동작하는 부트 시퀀스가 만들어지지 않으며, RTS만으로는 다른 아키텍처로의 이식이나 표준 호환성을 보장할 수 없습니다. 세 계층이 협력해야 `int main(void)`의 첫 줄에서 전역 변수가 초기값을 가지고 있는 *결과*가 만들어지게 됩니다. 

`main` 함수 호출 전에 어떤 일이 일어나는지, 그리고 왜 진입점(entry point) 함수가 `_c_int00`인지에 대한 궁금증에서 시작한 공부가, C 표준에서 ABI를 거쳐 툴체인 구현까지 이어지는 규약 전체로 확장되었습니다. 깊게 파고들수록 배울 게 많아지는 것 같습니다. 긴 글 읽어주셔서 감사합니다.

---

*참고 자료: 
- ISO/IEC 9899:201x Committee Draft (N1570)
- TMS320C6000 Embedded Application Binary Interface (SPRAB89B)
- TMS320C6000 Assembly Language Tools User's Guide (SPRUI03)
- TMS320C6000 Optimizing Compiler User's Guide (SPRUI04)
- ARM/Hercules Optimizing Compiler User's Guide (SPRU514Y)
- ARM Optimizing Compiler User's Guide (SPNU151V)
- TI Arm Clang User's Guide 
- TI CGT  `lib/src/boot.c`, `args_main.c`.*
