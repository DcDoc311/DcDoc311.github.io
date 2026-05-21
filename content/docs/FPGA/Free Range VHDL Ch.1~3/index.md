---
title: Free Range VHDL Ch.1~3
date: 2026-05-21
categories:
  - FPGA
tags:
  - architecture
  - entity
  - syntax
  - dataType
  - keywords
---
## 서문

소프트웨어 개발자가 처음 FPGA를 접할 때 가장 많이 하는 말이 있습니다.

> *"VHDL이 그냥 C랑 비슷한 거 아닌가요?"*

 **아닙니다.** VHDL은 C와 문법 구조가 유사해 보이지만 **근본적으로 다른 패러다임** 위에 서 있습니다. 이 차이를 이해하지 못하면 코드는 컴파일되지만 회로는 동작하지 않는 상황을 반복하게 됩니다.

이 글은 *Free Range VHDL* 교재의 챕터 1~3을 기반으로, VHDL이 무엇인지, 어떤 규칙 위에서 동작하는지, 그리고 실제 코드는 어떤 구조로 작성하는지에 대해 정리했습니다. 

---
## 1. VHDL이란 무엇인가 (Chapter 1)

### 1.1 VHDL 이란

**VHDL** 은 **V**HSI**C** **H**ardware **D**escription **L**anguage의 약자입니다. VHSIC은 또 다시 Very High-Speed Integrated Circuit(초고속 집적 회로)의 약자입니다. 이름 자체가 이 언어의 본질을 말해줍니다. 다시 말해, 하드웨어를 *기술(describe)* 하는 언어입니다.

VHDL은 1980년대 미국 국방부 주도로 개발되어 이후 IEEE 표준(IEEE 1076)으로 자리잡았습니다. 현재는 Xilinx(AMD), Intel(Altera) 등 모든 주요 FPGA 벤더의 합성 도구에서 지원하는 산업 표준 언어입니다.

### 1.2 소프트웨어 언어 vs 하드웨어 기술 언어

VHDL을 제대로 이해하려면 먼저 소프트웨어 언어와의 근본적인 차이를 명확히 해야 합니다.

**소프트웨어 언어(C, Java, Python 등)의 동작 방식:**

```
명령1 실행 → 완료
명령2 실행 → 완료
명령3 실행 → 완료
...
```

프로세서는 명령을 위에서 아래로 한 번에 하나씩 처리합니다. 이것이 **순차 실행(sequential execution)** 입니다.

**VHDL의 동작 방식:**

```
명령1 ─┐
명령2 ─┼─ 모두 동시에 실행
명령3 ─┘
```

VHDL에서는 architecture 본체에 작성된 문장들이 **모두 동시에(concurrently)** 실행됩니다. 이것은 비유가 아닙니다. 실제 하드웨어에서 AND 게이트, OR 게이트, 플립플롭이 물리적으로 동시에 동작하는 것처럼, VHDL 문장도 동시에 실행됩니다. (여기서 architecture는 실행 코드라 생각하시면 됩니다. 자세한 내용은 아래 3장에서 다루겠습니다.) 

이 차이를 직관적으로 이해하는 가장 좋은 방법은 간단한 논리 회로를 생각해보는 것입니다. 3개의 게이트가 있는 회로에서, 각 게이트의 입력이 바뀌면 그 게이트의 출력이 바뀝니다. 이때 "게이트 1이 먼저 계산되고 게이트 2가 나중에 계산된다"는 개념 자체가 성립하지 않습니다. 모든 게이트는 물리 법칙에 따라 동시에 반응합니다. VHDL은 바로 이 현상을 코드로 표현한 것입니다. 

### 1.3 VHDL의 두 가지 목적

VHDL은 두 가지 핵심 목적을 가집니다:

**① 모델링(Modeling):** 디지털 회로의 동작을 텍스트로 기술합니다. 어떤 입력에 어떤 출력이 나오는지, 어떤 상태 전이가 일어나는지를 명확하게 표현합니다.

**② 합성(Synthesis):** 작성된 VHDL 코드를 실제 하드웨어로 변환합니다. 합성기(synthesizer)라고 불리는 도구가 코드를 분석하여 FPGA 내부의 LUT(Look-Up Table), 플립플롭, DSP 블록 등의 물리적 자원에 매핑합니다.

이 두 목적 때문에 VHDL 코드를 작성할 때는 항상 두 가지 질문을 동시에 해야 합니다:
- *"이 코드가 의도한 동작을 시뮬레이션에서 올바르게 나타내는가?"*
- *"이 코드가 합성기에 의해 올바른 하드웨어로 변환될 것인가?"*

### 1.4 필수 규칙 : VHDL 코딩 전 반드시 기억할 두 가지

교재에서는 VHDL 개발 시 절대 잊지 말아야 할 두 가지 필수 규칙을 강조합니다.

**필수 규칙 1: VHDL은 하드웨어 설계 언어다**

VHDL 코드를 작성할 때는 **"프로그래밍한다"** 는 생각을 버려야 합니다. 대신 **"하드웨어를 설계한다"** 는 관점으로 접근해야 합니다. 특정 구문(process) 안에 있지 않는 한, 작성하는 모든 코드 라인은 동시에 실행됩니다.

만약 작성한 VHDL 코드가 C나 Java 코드처럼 보인다면, 그것은 나쁜 VHDL 코드일 가능성이 높습니다. 고수준 언어 스타일로 작성된 VHDL 코드는 합성기가 전혀 의도하지 않은 회로를 만들거나, 아예 합성에 실패하거나, 동작하더라도 불필요하게 복잡하고 비효율적인 하드웨어를 만들어냅니다.

**필수 규칙 2: 하드웨어의 구조를 머릿속에 그려라**

VHDL을 작성하기 전에 만들고자 하는 회로가 어떤 형태인지 대략적으로 그릴 수 있어야 합니다. VHDL이 강력하긴 하지만, 기본 디지털 구조를 이해하지 못한 채로 코드만 작성하면 원하는 회로를 만들 수 없습니다.

아무리 복잡한 디지털 회로도 결국 기본 구성 요소들의 조합입니다. (AND/OR/NOT 게이트, 멀티플렉서, 플립플롭, 카운터, FSM 등) 이 구성 요소들이 어떻게 연결되어야 하는지 머릿속에 그릴 수 있을 때 비로소 VHDL 코드가 의미를 가집니다.

### 1.5 VHDL 개발 도구

VHDL 기반 시스템의 구현은 크게 네 단계를 거칩니다:

| 단계 | 내용 | 대표 도구 |
|------|------|-----------|
| **코드 작성** | VHDL 소스 파일 작성 | 텍스트 에디터, IDE |
| **컴파일** | 문법 오류 확인 | Vivado, Quartus |
| **시뮬레이션** | 기능 검증 | Vivado Simulator, ModelSim |
| **합성 & 구현** | 실제 FPGA 비트스트림 생성 | Vivado, Quartus |

FPGA 개발에서 가장 많이 사용되는 도구는 Xilinx(현 AMD)의 **Vivado**와 Intel(구 Altera)의 **Quartus Prime**입니다. 두 도구 모두 무료 버전을 제공하며, 기본적인 FPGA 개발에는 무료 버전으로 충분합니다. FPGA 글의 모든 예제는 Vivado를 통해 작성되었습니다. 

---
## 2. VHDL 문법 (Chapter 2)

코딩을 시작하기 전에 반드시 알아야 할 VHDL의 기본 문법 규칙들이 있습니다. 이 규칙들은 암기가 필요한 수준의 내용입니다. 

### 2.1 대소문자 구분 없음

VHDL은 대소문자를 구분하지 않습니다(case-insensitive). 다음 두 문장은 완전히 동일합니다:

```vhdl
Dout <= A and B;
doUt <= a AnD b;  -- 완전히 동일
```

이것이 좋은 코딩 스타일은 아닙니다. 하지만 VHDL 컴파일러가 대소문자를 구분하지 않는다는 사실은 알고 있어야 합니다.

실무에서 권장하는 컨벤션:
- **예약어(keywords):** 소문자 (`entity`, `architecture`, `begin`, `end` 등)
- **상수:** 대문자 (`CLK_FREQ`, `MAX_COUNT`)
- **신호/변수:** 소문자 + 밑줄 (`rx_data_valid`, `clk_80mhz`)

### 2.2 공백 무시

VHDL 컴파일러는 공백(스페이스, 탭)을 무시합니다. 아래 두 문장은 동일합니다:

```vhdl
nQ <= In_a or In_b;
nQ    <=in_a OR    in_b;  -- 동일
```

공백은 컴파일러에게 무의미하지만, **본인과 동료에게는 매우 중요합니다.** 일관된 들여쓰기(보통 4칸)를 사용하는 것이 코드 가독성에 큰 영향을 미칩니다.

### 2.3 주석: `--`

VHDL의 주석은 `--`로 시작하며, 해당 줄 끝까지 적용됩니다:

```vhdl
-- 이 줄 전체가 주석입니다
signal clk_in : std_logic;  -- 인라인 주석도 가능

-- 블록 주석은 없음
-- 여러 줄이 필요하면 각 줄마다 -- 를 붙여야 함
```

C/C++의 `/* */` 같은 블록 주석이 VHDL에는 없습니다. 여러 줄에 걸친 주석이 필요하면 각 줄에 `--`를 붙여야 합니다.

**주석은 아끼지 말고 많이 사용하세요.** 특히 다음 상황에서 주석은 필수입니다:
- 신호의 목적과 의미가 이름만으로 명확하지 않을 때
- FSM 상태 전이 조건
- 타이밍 제약 관련 코드
- CDC 경계를 넘는 신호 처리

### 2.4 괄호 활용

VHDL에는 연산자 우선순위 규칙이 있지만, 이를 모두 외우는 것보다 **괄호를 적극적으로 사용하는 것**이 훨씬 현명합니다:

```vhdl
-- 모호한 표현 (and가 먼저인지 or가 먼저인지 불명확)
if x = '0' and y = '0' or z = '1' then

-- 명확한 표현 (의도가 코드에 드러남)
if ( ((x = '0') and (y = '0')) or (z = '1') ) then
```

코드의 독자는 우선순위 규칙을 외우고 있지 않을 수 있습니다. 괄호를 통해 의도를 명확하게 전달하는 것이 좋은 VHDL 스타일입니다.

### 2.5 문장의 끝: 세미콜론 `;`

모든 VHDL 문장은 세미콜론으로 끝납니다. 초보자가 가장 많이 하는 실수 중 하나가 세미콜론 누락입니다.

특히 다음 패턴들을 주의해야 합니다:

```vhdl
-- if 문: end if 뒤에 세미콜론
if (A = '1') then
    Y <= '1';
end if;          -- ← 세미콜론 필수

-- case 문: end case 뒤에 세미콜론
case SEL is
    when "00" => Y <= A;
    when others => Y <= '0';
end case;        -- ← 세미콜론 필수

-- entity 선언: 마지막 포트 뒤에는 세미콜론 없음
entity my_ckt is
port (
    A : in  std_logic;
    B : in  std_logic;
    F : out std_logic   -- ← 마지막 포트에는 세미콜론 없음!
);
end my_ckt;
```

2008년 이후 VHDL에서 entity의 마지막 포트에 세미콜론을 사용하여도 컴파일상 문제가 없습니다. 하지만, 호환성과 관례 때문에 마지막 포트에는 세미콜론을 사용하지 않는 것이 좋습니다. 
### 2.6 if, case, loop 문법 요약

VHDL을 처음 배울 때 가장 많이 실수하는 부분입니다. 

| 구문        | 올바른 형태                                | 주의사항                    |
| --------- | ------------------------------------- | ----------------------- |
| `if`      | `if ... then ... end if;`             | `else if` X → `elsif` O |
| `case`    | `case ... is ... end case;`           | `when others` 항상 포함 권장  |
| `loop`    | `loop ... end loop;`                  | `for`, `while` 루프도 동일   |
| `process` | `process(...) begin ... end process;` | 파라미터 항목 주의              |

특히 **`elsif`** 는 매우 중요합니다. `else if`라고 쓰면 컴파일 오류가 납니다:

```vhdl
-- 틀린 예
if (A = '1') then
    Y <= '1';
else if (B = '1') then  -- X 컴파일 오류!
    Y <= '0';
end if;

-- 올바른 예
if (A = '1') then
    Y <= '1';
elsif (B = '1') then    -- O elsif
    Y <= '0';
else
    Y <= 'Z';
end if;
```

### 2.7 식별자(Identifier) 명명 규칙

식별자란 신호, 변수, 포트, entity 등의 이름입니다. 좋은 식별자는 코드의 가독성을 크게 향상시킵니다.

**필수 규칙 (지키지 않으면 컴파일 오류):**

```vhdl
-- 유효한 식별자
data_bus
clk_in
rx_byte_cnt
WE              -- 단순하지만 관례적으로 널리 사용됨
div_flag

-- 유효하지 않은 식별자
3Bus_val        -- 숫자로 시작
mid_$num        -- 특수문자 포함 ($)
last__val       -- 연속 밑줄 (__)
str_val_        -- 밑줄로 끝남
in              -- 예약어
```

**권장 규칙 (안 지켜도 컴파일은 되지만 나쁜 코드):**

```vhdl
-- 나쁜 식별자 (의미 불명확)
signal x1   : std_logic;
signal temp : std_logic;
signal DDD  : std_logic_vector(7 downto 0);

-- 좋은 식별자 (의미가 이름에서 드러남)
signal rx_data_valid  : std_logic;
signal spi_clk_div    : std_logic_vector(7 downto 0);
signal fifo_wr_en     : std_logic;
```

> 신호 이름에 방향, 도메인, 극성을 반영하는 것이 좋습니다. 예를 들어 `cs_n`은 칩 셀렉트(CS)이고 active-low(`_n`)라는 것을 이름에서 바로 알 수 있습니다. `clk_80mhz`는 80MHz 클럭 도메인임을 명시합니다. 이런 컨벤션은 특히 CDC 디버깅 시 큰 도움이 됩니다.

### 2.8 예약어(Reserved Words)

다음 단어들은 VHDL에서 특별한 의미를 가지므로 식별자로 사용할 수 없습니다:

```
access   after    alias    all      architecture  array
begin    body     buffer   bus      case          component
constant downto   else     elsif    end           entity
for      function generate generic  if            in
inout    is       library  loop     map           mod
not      of       others   out      package       port
process  range    record   rem      report        return
select   signal   then     to       type          use
variable wait     when     while    with          xor
and      or       nand     nor      xnor          not
```

---

## 3. VHDL 설계 단위 (Chapter 3)

VHDL 코드의 기본 구조는 **entity**와 **architecture**의 쌍으로 이루어집니다. 이 두 개념을 이해하는 것이 VHDL 코딩의 출발점입니다.

### 3.1 블랙박스 접근법과 계층 설계

VHDL은 **블랙박스(black-box)** 접근법을 기반으로 합니다. 복잡한 시스템을 설계할 때, 전체를 한꺼번에 이해하려 하지 않고 계층적으로 분해합니다.

각 블랙박스는 두 가지 관점에서 볼 수 있습니다:
1. **외부 관점 (인터페이스):** 무슨 입력이 들어가고 무슨 출력이 나오는가? → **entity**
2. **내부 관점 (구현):** 그 입출력 관계가 어떻게 구현되는가? → **architecture**

이 분리 덕분에 같은 entity(같은 인터페이스)에 대해 여러 개의 architecture(다른 구현)를 가질 수 있습니다. 예를 들어 곱셈기를 순차 방식으로 구현한 architecture와 파이프라인 방식으로 구현한 architecture를 같은 entity로 교체하며 테스트할 수 있습니다.

### 3.2 Entity: 회로의 인터페이스

**entity**는 회로의 입출력 포트를 정의하는 블랙박스 선언입니다.

```vhdl
entity <entity_name> is
port (
    <port_name> : <mode> <type>;
    <port_name> : <mode> <type>
    -- 마지막 포트에는 세미콜론 없음!
);
end <entity_name>;
```

**포트 모드(mode):**

| 모드 | 방향 | 설명 |
|------|------|------|
| `in` | →  entity | 입력 신호 |
| `out` | entity → | 출력 신호 |
| `inout` | ↔ | 양방향 신호 (예: 버스 데이터 라인) |

**예시:**

```vhdl
-----------------------------------------------------
-- killer_ckt: 사망 판정 회로 인터페이스 기술
-----------------------------------------------------
entity killer_ckt is
port (
    life_in1       : in  std_logic;      -- 생명 입력 1
    life_in2       : in  std_logic;      -- 생명 입력 2
    ctrl_a, ctrl_b : in  std_logic;      -- 제어 신호 (같은 타입이면 한 줄에)
    kill_a         : out std_logic;      -- 사망 출력 A
    kill_b, kill_c : out std_logic       -- 사망 출력 B, C
);
end killer_ckt;
```

**번들(Bundle): 여러 비트 신호**

관련된 여러 신호는 `std_logic_vector`로 묶어 표현합니다. 이것을 교재에서는 **번들(bundle)** 이라고 부릅니다:

```vhdl
entity mux4 is
port (
    a_data    : in  std_logic_vector(0 to 7);   -- 8비트 입력 번들
    b_data    : in  std_logic_vector(0 to 7);
    c_data    : in  std_logic_vector(0 to 7);
    d_data    : in  std_logic_vector(0 to 7);
    sel1, sel0 : in  std_logic;
    data_out   : out std_logic_vector(0 to 7)   -- 8비트 출력 번들
);
end mux4;
```

**`to` vs `downto`:**

벡터의 비트 방향을 지정하는 두 가지 방식이 있습니다:

```vhdl
signal A : std_logic_vector(7 downto 0);  -- A(7)이 MSB, A(0)이 LSB (권장)
signal B : std_logic_vector(0 to 7);      -- B(0)이 MSB, B(7)이 LSB
```

`downto` 방향이 일반적인 디지털 설계 관례(MSB가 왼쪽)와 일치하므로 특별한 이유가 없으면 `downto`를 사용하는 것이 좋습니다.

### 3.3 표준 라이브러리: IEEE std_logic_1164

VHDL에서 `std_logic` 타입을 사용하려면 반드시 IEEE 라이브러리를 선언해야 합니다. 거의 모든 VHDL 파일은 다음 세 줄로 시작합니다:

```vhdl
library IEEE;
use IEEE.std_logic_1164.all;  -- std_logic, std_logic_vector 타입 제공
use IEEE.numeric_std.all;      -- 산술 연산(unsigned, signed, +, -, *) 제공
```

**`std_logic` 타입의 9가지 값:**

| 값 | 의미 | 주로 사용하는 상황 |
|----|------|-------------------|
| `'0'` | 논리 0 | 일반 디지털 신호 |
| `'1'` | 논리 1 | 일반 디지털 신호 |
| `'Z'` | 고임피던스 | 3-상태 버스, tri-state 출력 |
| `'X'` | 알 수 없음 | 시뮬레이션에서 충돌 감지 |
| `'-'` | Don't care | 합성에서 최적화 허용 |
| `'U'` | 초기화되지 않음 | 시뮬레이션 초기 상태 |

실제 합성 코드에서는 `'0'`, `'1'`, `'Z'`를 주로 사용합니다.

> **주의:** `IEEE.std_logic_arith`와 `IEEE.std_logic_unsigned` 같은 라이브러리는 비표준 Synopsys 라이브러리입니다. 호환성 문제를 피하려면 항상 `IEEE.numeric_std`를 사용하세요.

### 3.4 architecture 회로의 구현

**architecture**는 entity가 실제로 어떻게 동작하는지를 기술합니다.

```vhdl
architecture <arch_name> of <entity_name> is
    -- 선언 영역: 내부 신호, 상수, 컴포넌트 선언
    signal <sig_name> : <type>;
    constant <const_name> : <type> := <value>;
begin
    -- 구현 영역: 병행 문장들 (모두 동시에 실행)
    <concurrent_statements>
end <arch_name>;
```

**완전한 예시 — AND 게이트:**

```vhdl
library IEEE;
use IEEE.std_logic_1164.all;

-- entity: 외부 인터페이스
entity and_gate is
port (
    A : in  std_logic;
    B : in  std_logic;
    F : out std_logic
);
end and_gate;

-- architecture: 내부 구현
architecture dataflow of and_gate is
begin
    F <= A and B;  -- 병행 신호 할당
end dataflow;
```

### 3.5 신호(Signal) vs 변수(Variable)

VHDL에서 값을 저장하는 두 가지 주요 객체 유형입니다.

**신호(Signal):**
- architecture 선언 영역에서 선언
- `<=` 연산자로 할당
- 할당 후 즉시 바뀌지 않고 **델타 지연(delta delay)** 후에 업데이트
- 하드웨어의 와이어(wire)에 해당
- 병행 영역과 프로세스 모두에서 사용 가능

**변수(Variable):**
- 프로세스(process), 함수(function), 프로시저(procedure) 내부에서만 선언
- `:=` 연산자로 할당
- 할당 **즉시** 업데이트
- 프로세스 내부의 중간 계산에 사용

```vhdl
architecture example of my_ckt is
    signal sig_1 : std_logic;  -- 신호 선언 (선언 영역)
begin
    process (A, B, C)
        variable var_1 : integer;  -- 변수 선언 (프로세스 내부)
    begin
        sig_1 <= A;          -- 신호 할당: 델타 지연 후 업데이트
        var_1 := 34;         -- 변수 할당: 즉시 업데이트
    end process;
    G <= not (A and B);      -- 병행 할당 (process와 동시 실행)
end example;
```

**차이점: 업데이트 시점**

```vhdl
-- 신호의 함정: 같은 프로세스 내에서 신호는 즉시 갱신되지 않음
process(CLK)
begin
    if rising_edge(CLK) then
        A <= B;       -- A에 B 할당
        C <= A;       -- 이때 A는 아직 이전 값! (새 B 값이 아님)
    end if;
end process;
-- 결과: A=B(새값), C=A(이전값) → 쉬프트 레지스터 동작

-- 변수의 즉시 갱신
process(CLK)
    variable tmp : std_logic;
begin
    if rising_edge(CLK) then
        tmp := B;     -- tmp에 B 할당 (즉시 반영)
        C   <= tmp;   -- 이때 tmp는 이미 새 B 값
    end if;
end process;
-- 결과: C=B(새값) → 단순 레지스터 동작
```

이 차이를 이해하지 못하면 FSM이나 파이프라인 설계 시 예상과 다른 동작을 경험하게 됩니다.

### 3.6 중간 신호(Intermediate Signal)

entity 출력으로 선언된 신호는 신호 할당 연산자의 **오른쪽**에 사용할 수 없습니다. 이 제약을 해결하는 방법이 **중간 신호** 선언입니다:

```vhdl
entity bad_example is
port (
    A, B : in  std_logic;
    F    : out std_logic
);
end bad_example;

-- 잘못된 예: out 포트를 오른쪽에 사용
architecture bad of bad_example is
begin
    F <= A and B;
    G <= F or A;   -- X F는 out 포트이므로 오른쪽에 사용 불가
end bad;

-- 올바른 예: 중간 신호 사용
architecture good of bad_example is
    signal F_int : std_logic;  -- 중간 신호 (모드 지정자 없음)
begin
    F_int <= A and B;
    F     <= F_int;            -- out 포트에 중간 신호 값 전달
    G     <= F_int or A;       -- 중간 신호는 오른쪽에 자유롭게 사용
end good;
```

### 3.7 세 가지 모델링 스타일

architecture를 작성하는 방법은 크게 세 가지입니다:

**① 데이터 흐름(Data-flow) 스타일:**
병행 신호 할당 문장으로 데이터가 회로를 통해 흐르는 방식을 직접 기술합니다. 조합 논리 회로에 적합합니다.

```vhdl
architecture dataflow of full_adder is
begin
    SUM  <= A xor B xor CIN;
    COUT <= (A and B) or (B and CIN) or (A and CIN);
end dataflow;
```

**② 동작(Behavioral) 스타일:**
프로세스(process) 문장을 사용하여 회로의 동작을 기술합니다. 순차 논리(플립플롭, FSM 등)를 표현하는 데 주로 사용됩니다.

```vhdl
architecture behavioral of d_ff is
begin
    process(CLK, RST)
    begin
        if (RST = '1') then
            Q <= '0';
        elsif rising_edge(CLK) then
            Q <= D;
        end if;
    end process;
end behavioral;
```

**③ 구조적(Structural) 스타일:**
컴포넌트(component)를 선언하고 인스턴스화하여 회로를 기술합니다. 계층 설계의 핵심입니다.

```vhdl
architecture structural of top_level is
    component and_gate is
        port (A, B : in std_logic; F : out std_logic);
    end component;
    signal int_sig : std_logic;
begin
    U1: and_gate port map (A => IN1, B => IN2, F => int_sig);
    U2: and_gate port map (A => int_sig, B => IN3, F => OUT1);
end structural;
```

실제 설계에서는 이 세 가지 스타일을 **혼합**하여 사용하는 것이 일반적입니다.

---
## 4. 정리

챕터 1~3의 내용을 종합하여 완전한 VHDL 코드를 작성해봤습니다. 

**목표:** 4:1 멀티플렉서를 구현하는 완전한 VHDL 모듈

```vhdl
architecture behavioral of mux4to1_8bit is
begin
    process(D0, D1, D2, D3, SEL)
    begin
        if (SEL = "11") then
            Y <= D3;
        elsif (SEL = "10") then
            Y <= D2;
        elsif (SEL = "01") then
            Y <= D1;
        else
            Y <= D0;
        end if;
    end process;
end behavioral;
```

- **library/use:** IEEE 표준 라이브러리 선언 (Ch.3)
- **entity:** 블랙박스 인터페이스 정의, `std_logic_vector` 번들 사용 (Ch.3)
- **downto:** MSB가 왼쪽인 방향 지정 (Ch.3)
- **architecture ... begin ... end:** 내부 구현 블록 (Ch.3)

---

## 마무리: 챕터 1~3 정리

| 개념                     | 핵심 내용                                     |
| ---------------------- | ----------------------------------------- |
| **VHDL의 본질**           | 하드웨어 기술 언어, 순차 실행이 아닌 **병행 실행**           |
| **필수 규칙 1**            | "프로그래밍"이 아니라 "하드웨어 설계"                    |
| **필수 규칙 2**            | 코드 전에 회로 구조를 머릿속에 그려라                     |
| **대소문자**               | 구분 없음. 하지만 일관된 컨벤션 사용 권장                  |
| **주석**                 | `--` 한 줄 주석만 있음. 많이 사용할수록 좋음              |
| **문장 끝**               | 항상 `;` 세미콜론                               |
| **식별자**                | 자기 설명적으로, 알파벳 시작, 연속 밑줄 금지                |
| **entity**             | 블랙박스의 외부 인터페이스(포트 선언)                     |
| **architecture**       | 블랙박스의 내부 구현(동작 기술)                        |
| **signal vs variable** | 신호는 델타 지연 후 갱신, 변수는 즉시 갱신                 |
| **중간 신호**              | out 포트를 오른쪽에 사용하려면 중간 신호 선언               |
| **세 가지 스타일**           | data-flow, behavioral, structural (혼합 가능) |

