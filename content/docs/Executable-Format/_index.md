---
title: Executable Format
---

Windows PE, ELF 등 실행 파일 포맷의 내부 구조와 로더 동작 방식을 다룹니다.  
링커 출력물이 어떻게 메모리에 적재되고 실행되는지를 분석합니다.

{{< callout type="info" >}}
주요 환경: Windows x86/x64, TI COFF/ELF, Visual Studio, PE-bear
{{< /callout >}}

## 다루는 주제

{{< cards >}}
  {{< card title="PE Format" subtitle="DOS Header, PE Header, Section, DataDirectory" icon="document-text" >}}
  {{< card title="ELF Format" subtitle="ELF Header, Program Header, Section Header" icon="document" >}}
  {{< card title="Linker" subtitle="링커 스크립트, 섹션 배치, 심볼 해석" icon="code" >}}
  {{< card title="Loader" subtitle="메모리 적재, Relocation, Import 해석" icon="server" >}}
{{< /cards >}}