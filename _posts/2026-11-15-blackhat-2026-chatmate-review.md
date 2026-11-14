---
title: "[Black Hat USA 2026] ChatMate: Remote Prompt Execution on AI Assistants through Sandbox Escaping 리뷰"
date: 2026-11-15 00:30:00 +0900
categories: [논문/컨퍼런스]
tags: [Black Hat, AI 보안, Prompt Injection, Sandbox Escape, Copilot, RPE]
---

## 1. 개요
- 컨퍼런스: Black Hat USA 2026 (2026.08.05~06, Las Vegas)
- 발표 제목: ChatMate: Remote Prompt Execution on AI Assistants through Sandbox Escaping
- 발표자: Ori Lahav (Rubrik Zero Labs), 공동 연구자 Dan Avraham
- 분야: AI 보안, 클라우드·컨테이너 보안
- 자료: [발표 영상](https://youtu.be/MspRSCAb6uI), [Rubrik Zero Labs 블로그](https://zerolabs.rubrik.com/blog/breaking-m365-copilot-sandbox-chatmate)

## 2. 한 줄 요약
이 발표는 Microsoft 365 Copilot을 대상으로, 악성 문서 하나로 피해자의 AI 어시스턴트 세션을 원격에서 조종할 수 있는 문제를 프롬프트 인젝션과 샌드박스 탈출을 연결하는 방식으로 공격했다.

## 3. 배경 및 문제 정의
- 최신 AI 어시스턴트는 텍스트 생성만 하지 않고, 스프레드시트 작성이나 문서 파싱, 연산을 위해 내부적으로 코드 인터프리터(Python)를 실행한다.
- 이 코드는 네트워크가 차단된 격리 샌드박스에서 실행되지만, 공격자가 제어하는 코드를 안전하게 실행하는 샌드박스를 만드는 것은 매우 어렵다.
- 발표자는 이 지점에 주목하여 새로운 공격 클래스인 Remote Prompt Execution(RPE)을 정의했다. RPE는 공격자가 피해자의 인증된 AI 어시스턴트에 프롬프트를 반복해서 입력하고, 출력을 확인한 뒤 다음 프롬프트를 조정할 수 있는 능력을 의미하며, 이 과정에 피해자의 추가 상호작용은 필요하지 않다.

## 4. 핵심 내용
### 공격 체인
1. **초기 실행**: 악성 Word 문서에 숨긴 지시로 간접 프롬프트 인젝션을 수행하고, LLM 안전 필터를 우회하여 샌드박스 안에서 비특권 코드 실행 권한을 얻는다.
2. **권한 상승**: 컨테이너 내부에서 root로 권한을 상승시켜, 일반 권한으로는 보이지 않던 공격 표면을 노출시킨다.
3. **샌드박스 탈출**: AKS Image Streaming 서비스의 인증 없는 내부 데몬을 악용하여 호스트에 임의 파일을 쓰고, 샌드박스를 완전히 벗어난다. 체인에는 path traversal, .toml 인젝션, LD_PRELOAD 익스플로잇이 사용되었다.
4. **프롬프트 셸 구성**: 탈출한 환경을 공격자와 피해자 세션을 잇는 양방향 채널로 바꾼다. 공격자는 Copilot의 응답 전체를 확인하고 다음 프롬프트를 입력하는 과정을 반복하면서 피해자의 M365 데이터를 읽는다.

### 핵심 아이디어
- 개별 버그는 LLM 취약점(1단계)과 컨테이너·인프라 취약점(2~3단계)으로 나뉘지만, 이를 하나의 체인으로 조합하여 "AI 세션 탈취"라는 새로운 결과를 만들었다.
- 유사 연구인 Varonis의 Reprompt는 의도된 인터넷 접근 기능을 악용한 반면, ChatMate는 어시스턴트의 실행 경계를 넘어서 통신 채널을 직접 만든다는 점에서 차별점이 있다.

### 결과
- CVE-2026-32193 (Critical)이 발급되었고, Microsoft로부터 버그 바운티 $48,000를 받았다.
- AKS의 path traversal 결함은 발표 전에 패치되었지만, RPE라는 공격 클래스 자체는 Copilot에만 국한되지 않으며, 유사한 컨테이너 기반 실행 환경을 쓰는 다른 AI 어시스턴트도 잠재적 대상이 된다.

## 5. 인상 깊었던 점
공격의 시작이 사용자가 숨겨진 자연어가 있는 파일을 LLM에 업로드하면서 시작되는데 이 부분이 사회공학과 유사하면서도 뚜렷한 차이도 느껴졌다. 기존의 사회공학은 사용자가 악성코드를 실행하게 되면서 공격의
시작했다면 그 중간에 LLM이 끼어든 느낌이다. 사실 사회공학은 단순하게 출처가 확실하지 않은 링크나 파일을 열지만 않으면 되는데... 이 취약점은 한번 더 포장하는 느낌.. 컨퍼런스 인트로에서 금요일 오후 일이 몰릴 때 업무 지시 파일인 척 보낸다고 했는데 방심할 수 밖에 없다. 단지 LLM에 파일을 맡기고 일을 부탁하기만 했는데 서버가 탈취될 거라고는 생각도 못할 것이다..


## 6. 한계 및 의문점
이 공격이 가능했던 이유 중 하나는 사용자가 copilot에게 부여했던 접근 권한에 있다. 접근이 가능한 영역 안에서 RPE로 공격자가 정보를 탈취한 것이다.
Ai어시스턴스를 쓰지 않을 때는 다시 권한을 뺐었다가 실행시에만 부여하도록 하면 되지않을까?

## 7. 내 공부·연구와의 연결
현재 RadixMend라는 프로젝트에서 SGLang의 radix tree 기반 KV 캐시를 다루고 있다. SGLang은 여러 요청이 같은 프롬프트 prefix를 공유하면 KV 캐시를 재사용하는데, 이때 캐시에 저장된 값은 별도의 검증 없이 그대로 신뢰된다.

ChatMate를 보면서 같은 문제를 보고 있다는 생각이 들었다. ChatMate에서 결정적인 역할을 한 것은 인증 없이 열려 있던 내부 데몬이었고, 이는 "내부 구성 요소끼리는 서로 믿어도 된다"는 가정이 깨진 사례다. RadixMend에서 다룬 KV 캐시도 "한 번 계산해서 저장한 내부 상태는 믿어도 된다"는 가정 위에 있다. 결국 AI 시스템의 취약점은 모델 자체보다, 시스템이 검증 없이 신뢰하는 내부 경계에서 나온다고 느꼈다.

### 시스템 해킹 공부
체인의 후반부인 권한 상승과 샌드박스 탈출은 전통적인 리눅스 시스템 해킹 기법으로 이루어져 있었다. 새로운 분야의 공격도 결국 시스템 해킹의 기본기로 완성된다는 점에서 현재 진행 중인 시스템 해킹 공부의 필요성을 더 느꼈다.

## 8. 참고 자료
- [Rubrik Zero Labs - Breaking the M365 Copilot Sandbox with ChatMate](https://zerolabs.rubrik.com/blog/breaking-m365-copilot-sandbox-chatmate)
- [Rubrik Zero Labs - Part One: Getting Root in the Code Interpreter](https://zerolabs.rubrik.com/blog/breaking-m365-copilot-sandbox-part-one-getting-root-code-interpreter)
- [CODE BLUE 2026 - 발표 소개](https://codeblue.jp/en/program/time-table/day1-t1-05-chatmate/)
- [CVE-2026-32193](https://dbugs.ptsecurity.com/vulnerability/PT-2026-47061)
- [Forkast - Remote Prompt Execution Is a New Vulnerability Class](https://forkast.news/remote-prompt-execution-is-a-new-vulnerability-class-chatmate-just-showed-how-it-works-on-copilot/)