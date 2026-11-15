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
이 발표에서 연구진은 프롬프트 인젝션과 샌드박스 탈출을 연결해, 악성 문서 하나로 피해자의 Microsoft 365 Copilot 세션을 원격에서 조종할 수 있음을 보였다.

## 3. 배경 및 문제 정의
- 최신 AI 어시스턴트는 텍스트 생성만 하지 않고, 스프레드시트 작성이나 문서 파싱, 연산을 위해 내부적으로 코드 인터프리터(Python)를 실행한다.
- 이 코드는 네트워크가 차단된 격리 샌드박스에서 실행되지만, 공격자가 제어하는 코드를 안전하게 실행하는 샌드박스를 만드는 것은 매우 어렵다.
- 발표자는 이 지점에 주목하여 새로운 공격 클래스인 Remote Prompt Execution(RPE)을 정의했다. RPE는 공격자가 피해자의 인증된 AI 어시스턴트에 프롬프트를 반복해서 입력하고, 출력을 확인한 뒤 다음 프롬프트를 조정할 수 있는 능력을 의미하며, 이 과정에 피해자의 추가 상호작용은 필요하지 않다.

## 4. 핵심 내용
### 공격 체인

#### 1. 초기 코드 실행: 간접 프롬프트 인젝션

공격자는 평범한 업무 문서처럼 보이는 Word 파일에 사람 눈에는 안 보이는 지시를 숨겨 둔다. 사용자가 이 문서를 Copilot에 올리면 Copilot이 숨은 지시를 읽고 코드 실행 도구를 호출한다.

그대로 코드를 보내면 Copilot의 안전 필터에 걸린다. 그래서 공격자는 Python 코드를 gzip으로 압축하고, 이것을 "압축 성능 테스트"로 꾸몄다. Copilot은 테스트라고 믿고 데이터를 풀어 안의 코드를 실행한 뒤 결과까지 돌려준다. 이 시점에서 공격자는 격리된 샌드박스 안에서 원하는 Python 코드를 실행할 수 있다.

#### 2. 샌드박스 안에서 root 얻기

샌드박스 코드는 일반 사용자 `ubuntu` 권한으로 돈다. 그런데 컨테이너의 첫 프로세스(PID 1)로 실행되는 `/app/entrypoint.sh`에 일반 사용자도 쓸 수 있었다.

Bash는 스크립트를 한 번에 다 읽지 않고 실행하면서 차례로 읽는다. 연구진은 이 점을 노려, 이미 실행 중인 스크립트의 `wait` 명령 뒤에 자기 코드를 덧붙였다. 그다음 기존 하위 프로세스를 종료해 `wait`가 끝나게 만들자, 덧붙인 코드가 root 권한으로 실행됐다. 다만 서비스가 죽은 걸 플랫폼이 알아채면 샌드박스를 새로 만들어 버리기 때문에, 종료한 서비스를 곧바로 다시 띄워야 했다.

root를 얻은 뒤 연구진은 root로 명령을 받아 실행하는 데몬을 하나 띄워 두고, 이후 작업을 계속 관리자 권한으로 이어갔다.

#### 3. 샌드박스 탈출과 호스트 RCE

샌드박스 안 8578번 포트에는 인증 없이 접근할 수 있는 Azure Container Runtime(acr) 데몬이 열려 있었다. 이 데몬의 `/config` API는 설정 이름(`ns`)을 그대로 파일 경로로 썼고, 경로 검사가 허술해 디렉터리 트래버설이 가능했다.

데몬은 받은 설정 이름으로 디렉터리를 만들고 그 안에 `hosts.toml`을 쓴다. 파일이 이미 있으면 쓰기가 실패하는데, 연구진은 이 실패 여부로 호스트의 특정 경로에 파일이 있는지 알아냈다. 그렇게 확인해 보니 데몬은 컨테이너 안이 아니라 **Kubernetes 노드의 호스트 파일 시스템에 root 권한으로** 파일을 쓰고 있었다.

연구진은 여기에 두 가지 기법을 더해, 호스트의 원하는 파일을 원하는 내용으로 쓸 수 있게 만들었다.

- **심볼릭 링크**: 데몬이 `hosts.toml`을 쓸 자리에 미리 심볼릭 링크를 만들어 두고 호스트의 목표 파일을 가리키게 했다. 데몬은 링크를 따라가 목표 파일을 덮어썼다.
- **TOML 인젝션**: 설정 이름에 들어간 개행 문자를 걸러내지 않았다. 개행을 넣어 고정된 TOML 템플릿의 `server` 항목 밖으로 빠져나가면, 원하는 내용을 파일에 덧붙일 수 있었다.

마지막 목표는 `/etc/ld.so.preload`였다. 이 파일에 등록한 공유 라이브러리는 노드에서 새로 시작하는 모든 동적 링크 프로세스에 함께 로드된다. 연구진의 라이브러리는 자신이 root 프로세스에 들어갔는지 확인한 다음 별도 프로세스로 떨어져 나와 동작했다. 연구진은 이렇게 Kubernetes 노드에서 root 권한으로 코드를 실행하는 데 성공했다.

#### 4. 양방향 채널과 Remote Prompt Execution

샌드박스는 인터넷이 막혀 있었지만 탈출한 노드는 밖과 통신할 수 있었다. 연구진은 샌드박스와 노드가 같이 쓰는 `/mnt/data` 볼륨을 통로로 삼았다. 샌드박스가 볼륨에 명령을 적으면, 노드에 심어 둔 root 라이브러리가 읽어서 실행하고 결과를 다시 볼륨에 적는다. 노드는 외부의 공격자 서버와도 연결되니, 결국 네트워크 격리를 건너뛰는 명령·응답 통로가 생긴 셈이다.

연구진은 이 통로를 프롬프트 전달에 썼다. 공격자가 밖에서 프롬프트를 보내면 통로를 거쳐 Copilot이 실행한 프로그램의 출력으로 들어간다. 문서에 숨긴 지시가 "프로그램 출력에 담긴 내용을 다음 지시로 따르라"고 해 두었기 때문에, Copilot은 이것을 피해자의 인증된 세션과 권한으로 처리한다. 응답은 같은 길을 거꾸로 타고 공격자에게 돌아간다.

공격자는 응답을 확인한 뒤 다음 프롬프트를 정해 다시 보낼 수 있다. 이걸 되풀이하면 피해자가 더 손대지 않아도 Copilot과 계속 대화하면서, 피해자의 Microsoft 365 권한으로 볼 수 있는 메일·파일·일정·채팅을 들여다볼 수 있다. 연구진은 이 양방향 프롬프트 제어 구조를 **ChatMate**라고 이름 붙였다.

### 핵심 아이디어
- 개별 버그는 LLM 취약점(1단계)과 컨테이너·인프라 취약점(2~3단계)으로 나뉜다. 연구진은 이를 하나의 체인으로 엮어 "AI 세션 탈취"라는 새로운 결과를 만들었다.
- 유사 연구인 Varonis의 Reprompt는 의도된 인터넷 접근 기능을 악용한 반면, ChatMate는 어시스턴트의 실행 경계를 넘어서 통신 채널을 직접 만든다는 점에서 차별점이 있다.

### 결과
- 연구진은 이 취약점으로 CVE-2026-32193(Critical)을 발급받았고, Microsoft에서 버그 바운티 $48,000를 받았다.
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