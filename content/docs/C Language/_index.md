---
title: C Language
---

C 언어의 동작 원리를 표준 명세 기반으로 다룹니다. 
</br>언어 설계, 메모리 모델, 컴파일러와의 관계 등 저수준 관점에서 접근합니다.

{{< callout type="info" >}}
주요 참조: C89 / C99 / C11 표준, N1570 Committee Draft, 컴파일러 구현체(GCC, TI CGT)
{{< /callout >}}

## 다루는 주제

{{< cards >}}
{{< card title="언어 표준과 추상 기계" subtitle="Abstract Machine, As-if Rule, Observable Behavior" icon="chip" >}}
{{< card title="타입 시스템" subtitle="Integer Promotion, Type Qualifier, Alignment" icon="cog" >}}
{{< card title="메모리와 객체" subtitle="Storage Duration, Linkage, Scope" icon="lightning-bolt" >}}
{{< card title="미정의 동작" subtitle="Undefined / Unspecified / Implementation-defined Behavior" icon="exclamation" >}}
{{< card title="전처리와 번역" subtitle="Translation Phases, Macro, Conditional Compilation" icon="document-text" >}}
{{< card title="표준 라이브러리" subtitle="libc, 런타임 초기화, 시스템 인터페이스" icon="library" >}}
{{< /cards >}}