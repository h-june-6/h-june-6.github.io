---
title: "[논문 리뷰] Crashing Through Defenses: Exploiting Segfaults and Chaining around Intel CET"
date: 2026-11-17 17:30:00 +0900
categories: [논문/컨퍼런스]
tags: [paper-review, sfop, intel-cet, srop, code-reuse, ieee-sp]
---

-   **저자:** Marcos Bajo, Ritvik Goyal, Apostolos Chatzianagnostou,
    Christian Rossow
-   **학회:** IEEE Symposium on Security and Privacy (IEEE S&P), 2026
-   **분야:** 시스템 보안, 코드 재사용 공격, 제어 흐름 보호
-   **논문:** [Crashing Through Defenses: Exploiting Segfaults and
    Chaining around Intel
    CET](https://syssec.cispa.de/papers/sfop-ieeesp2026.pdf)

## 0. 읽기 전 정리

이 논문은 Intel CET가 적용된 Linux 환경에서 메모리 손상 취약점을 이용해 
공격을 이어가는 SFOP를 제안한다. 
이름처럼 프로그램에서 발생하는 Segmentation Fault를 공격 과정에 활용하는 기법이다.

앞에서 언급한 CET는 ROP와 같은 코드 재사용 공격을 막기
위해 도입된 하드웨어 보안 기능이다.

CET가 활성화된 환경에서 공격자가
어떻게 실행 흐름을 조작할 수 있었을까? 저자들은 CET 기능
자체보다 Signal Handler을 이용했다.
Signal Handler는 프로그램이 오류로 인해 비정상적인 상황에 빠졌을 때 운영체제가 처리하는 과정이고
이 signal 중 하나가 SIGSEGV이다.

SIGSEGV는
프로그램이 접근해서는 안 되는 메모리 주소에 접근하면 발생하는 오류이다.

예를 들어

```c
int *ptr = NULL;
*ptr = 10;
```

이런 코드에서 ptr이 가리키는 주소는 NULL인데 여기에 값을 쓰려고 하면 일반적으로 SIGSEGV가 발생한다.

근데 이 논문에서는 이러한 SIGSEGV를 일부로 잘못된 메모리 접근을 하여 의도적으로 발생시킨다.

그럼 왜 시그널 핸들러를 발생시키려는걸까?

Linux의 시그널 처리 과정을 보면 다음과 같다.

```text
잘못된 메모리 접근
        ↓
SIGSEGV 발생
        ↓
커널이 CPU 실행 상태를 sigframe에 저장
        ↓
Signal Handler 실행
        ↓
sigreturn을 통해 기존 실행 상태 복구
```

여기서 핵심은 마지막이다.

커널이 sigframe에 저장된 정보를 바탕으로 CPU의 실행 상태를 복구한다는 점이다.
이 복구 과정을 SFOP에서 이용하려고 SIGSEGV를 발생시켜 Signal Handler를 실행하도록 만든 것이다.

기존 CET 우회 연구 중에는 C++ 가상 함수나 코루틴처럼 프로그램 내부의
특정 구조를 이용하는 방식이 있었다. 이런 방법은 해당 구조가 없는
프로그램에 그대로 적용하기 어렵다. SFOP는 Linux의 시그널 처리 기능을
이용해 공격을 연결한다는 점에서 다른 접근을 취한다.

## 1. 배경과 문제 정의

### 1-1. 연구 배경

Intel CET는 프로그램의 실행 흐름이 공격자에 의해 변조되는 것을 어렵게
만드는 기술이다. 주요 기능은 Shadow Stack과 IBT다. 
Shadow Stack은 함수가 반환할 때 일반 스택에 저장된 반환
주소와 별도로 보관된 주소를 비교한다. 공격자가 반환 주소를 바꾸면 두
값이 달라져 이를 탐지한다. IBT는 간접 호출이나 점프가 허용된 진입점으로
향하는지 확인한다.

저자들이 주목한 부분은 CET와 Linux의 Signal Handling이 함께 동작하는
과정이다. Signal은 프로세스에 특정 사건이 발생했음을 알리는 운영체제
기능이다. 예를 들어 잘못된 메모리 접근으로 SIGSEGV가 발생하면 운영체제는
등록된 Signal Handler를 실행한다. 이 과정에서는 중단된 프로그램의 실행
상태를 저장하고, Handler가 끝난 뒤 `rt_sigreturn`을 통해 실행 상태를
복원한다.

논문은 이 시그널 처리 과정과 CET의 보호 기능 사이에서 공격자가 이용할 수
있는 지점을 찾아낸다.

### 1-2. 위협 모델

논문에서 SFOP는 이미 메모리 손상 취약점을 확보한 공격자가 사용하는 후속
공격 기법이다. 공격자는 메모리 정보를 파악하거나 메모리 내용을 변조할 수
있고, 포인터를 조작할 수 있다고 가정한다. CET와 DEP/NX, Full RELRO 등
여러 방어 기법이 적용된 환경에서도 공격이 가능한지를 분석한다.

## 2. 핵심 아이디어와 방법론

### 2-1. SFOP의 동작 원리

SFOP는 Segmentation Fault를 반복적으로 이용해 공격 단계를 연결하는 코드
재사용 공격이다. 일반적으로 Segmentation Fault가 발생하면 프로세스가
종료되지만, Signal Handler가 등록되어 있으면 해당 Handler가 실행된다.
SFOP는 이 동작을 공격에 이용한다.

공격 흐름은 다음과 같다.

1.  메모리 손상 취약점을 이용해 필요한 포인터나 데이터를 변조한다.
2.  `sigaction()`으로 SIGSEGV Handler를 등록한다.
3.  의도적으로 오류를 발생시켜 SIGSEGV를 일으킨다.
4.  Signal Handler와 Signal Frame 처리 과정을 이용해 실행 상태를
    조작한다.
5.  `rt_sigreturn`을 통해 레지스터와 실행 위치를 복원하고 다음 공격
    단계로 넘어간다.
6.  필요한 경우 이 과정을 반복해 여러 함수를 연결한다.

Signal Frame은 시그널이 발생했을 때 중단된 프로그램의 실행 상태를
저장하는 자료구조다. `rt_sigreturn`은 저장된 정보를 이용해 실행을
재개하는 과정에 관여한다. 저자들은 이 기능이 CET와 함께 동작하는 방식을
분석해 공격 체인을 구성한다.

### 2-2. fSFOP와 bSFOP

논문은 SFOP를 fSFOP와 bSFOP 두 방식으로 나누어 설명한다.

fSFOP는 SIGSEGV가 발생하고 Signal Handler가 실행되는 과정을 반복적으로
활용한다. 이 과정에서 Early Kill과 Sigframe Pivoting 등의 기법을
사용한다. 시그널 처리 과정에서 공격자가 준비한 실행 상태로 이동한 뒤
다음 함수를 실행하도록 만드는 방식이다.

bSFOP는 Shadow Stack을 공격 흐름에 이용한다. 논문은 Custom Restorer와
Shadow Stack Pointer(SSP) 관련 동작을 분석하고, 이를 바탕으로 Recursive
Shadow Stack을 구성하는 방법을 제시한다. fSFOP와 bSFOP는 모두 Linux의
시그널 처리 과정을 이용하지만, 공격 체인을 구성할 때 활용하는 지점은
서로 다르다.

### 2-3. 기존 공격과의 비교

| 공격 기법 | 주요 아이디어 또는 활용 요소 |
|---|---|
| ROP | 기존 코드 조각을 연결해 실행 흐름 구성 |
| SROP | Signal Frame을 이용해 레지스터와 실행 상태 조작 |
| COOP | C++ 객체와 가상 함수 호출 구조 활용 |
| FOP | 프로그램에 존재하는 함수를 연결해 악의적인 동작 구성 |
| CFOP | C++ Coroutine의 실행 구조 활용 |
| SFOP | Linux Signal Handling과 CET의 상호작용 활용 |

SFOP는 특정 C++ 언어 구조 대신 Linux의 시그널 처리 과정을 이용한다. 이
점이 기존 기법과 비교했을 때 가장 눈에 띄는 차이다.

## 3. 실험과 평가, 한계

### 3-1. 실험

저자들은 여러 PoC를 통해 제안한 공격을 검증하고, 실제 프로그램에도 적용
사례를 제시한다. 실제 프로그램 사례로는 Nginx와 Ladybird가 등장한다.

Nginx 사례에서는 Nginx 1.4.0의 CVE-2013-2028 취약점을 이용한다. HTTP
요청 처리 과정에서 발생하는 메모리 손상 취약점을 출발점으로 삼아 fSFOP를
적용하고, 명령 실행으로 이어질 수 있음을 보인다.

Ladybird 사례에서는 CVE-2021-4327을 이용한다. 메모리 손상 취약점으로
공격 조건을 만든 뒤 fSFOP를 적용한다. 두 사례는 서로 다른 프로그램에서도
같은 공격 아이디어가 적용될 수 있음을 보여준다.

두 사례를 읽으면서 눈에 띈 점은 실제 프로그램에 적용한 공격이 fSFOP라는
것이다. bSFOP는 별도의 PoC를 통해 검증한 결과와 구분해서 평가할 필요가
있다.

### 3-2. 방어와 패치

저자들은 Linux 커널의 시그널 처리 과정에서 SFOP가 이용하는 동작을
제한하는 방어책도 논의한다. 핵심은 Signal Handler 등록과 Signal Frame
복원 과정에서 공격자가 실행 상태나 Shadow Stack을 임의로 조작하지
못하도록 검증을 강화하는 것이다. 논문은 Custom Restorer 검증,
`rt_sigreturn` 처리 과정의 검증, Signal Frame 관련 보호 등을 다룬다.

저자들의 평가에서 제안한 패치는 bSFOP를 차단하지만, fSFOP는 일부
완화하는 데 그친다. fSFOP를 완전히 막으려면 추가적인 커널 수준의 보호가
필요하다고 설명한다. 패치가 두 공격 방식에 동일한 효과를 내지 못한다는
점이 흥미로웠다.

### 3-3. 한계와 아쉬운 점

아쉬웠던 부분은 실제 프로그램에 적용한 사례가 제한적이라는 점이다.
Nginx와 Ladybird에서 공격 가능성을 보였지만, 이를 근거로 다양한 Linux
프로그램에서도 같은 조건으로 공격할 수 있다고 단정하기는 어렵다.
프로그램마다 메모리 구조와 시그널 처리 방식, 이용할 수 있는 취약점이
다르기 때문이다.

공격 성공률이나 실행 시간, 시스템 성능에 미치는 영향을 더 폭넓게
비교했다면 실제 환경에서 공격이 얼마나 현실적인지 판단하는 데 도움이
되었을 것 같다. bSFOP도 실제 프로그램에 적용한 사례가 추가된다면
fSFOP와의 차이를 더 분명하게 이해할 수 있을 것이다.

## 4. 느낀 점과 배운 점

ROP를 처음 접할 때도 이런 생각을 어떻게 해내지? 하는 의문이 들 정도로 
내 입장에선 새로운 패러다임이다.. 라는 느낌이였는데 이 논문도 비슷한 충격을
주었다. 프로그램의 예외 상황을 처리하는 기능으로만
생각했던 시그널 처리가 공격 표면이 될 수 있다는 점이 흥미로웠다.

그동안 시스템 해킹을 공부하면서는 주로 스택이나 힙의 메모리 구조, 반환
주소 변조처럼 프로그램 내부의 메모리 조작에 집중했다. 이번 논문을 읽고
나서는 공격자가 메모리를 변조한 다음 운영체제가 그 상태를 처리하는
과정까지 공격에 활용할 수 있다는 점을 알게 됐다. 앞으로는 취약한 코드뿐
아니라 커널이 사용자 공간의 실행 상태를 저장하고 복원하는 방식도 함께
공부해 보고 싶다.

## 5. 앞으로 할 것

### 5-1. 더 읽어볼 논문

이번 논문과 직접 연결되는 SROP, COOP, CFI 관련 연구를 이어서 읽어볼
계획이다.

-   Yingjie Guo, Liwei Chen, and Gang Shi, "Function-Oriented
    Programming: A New Class of Code Reuse Attack in C Applications,"
    IEEE Conference on Communications and Network Security (CNS), 2018.
    함수 단위의 코드 재사용 공격을 SFOP와 비교해 보고 싶다.
-   Marcos Bajo and Christian Rossow, "Await() a Second: Evading Control
    Flow Integrity by Hijacking C++ Coroutines," USENIX Security
    Symposium, 2025. 코루틴을 이용한 제어 흐름 보호 우회와 SFOP의 차이를
    확인하고 싶다.

### 5-2. 다른 분야로의 확장

후속 연구로는 ARM64 환경에서 비슷한 문제가 발생하는지 확인해 볼 수 있을
것 같다. ARM에는 PAC와 BTI처럼 포인터와 제어 흐름을 보호하는 기능이
있다. Intel CET와 구현 방식은 다르지만, 하드웨어 보호 기능과 운영체제의
예외 처리 과정이 만나는 지점을 비교하면 의미 있는 연구가 될 것 같다.

### 5-3. 재현해 보고 싶은 부분

논문 저자들이 공개한 [SFOP GitHub
저장소](https://github.com/signal-sfop/sfop)를 이용해 fSFOP와 bSFOP의
차이를 직접 확인해 보고 싶다.

처음부터 실제 프로그램의 취약점을 재현하기보다는, 먼저 저장소에서
제공하는 PoC로 Signal Handler와 Signal Frame이 어떻게 연결되는지 확인할
계획이다. 그다음 CET의 Shadow Stack이 활성화된 상태에서 공격 흐름이
어떻게 달라지는지 비교해 보고 싶다. 

### 5-4. 연구 방향

이 논문을 읽고 운영체제의 예외 처리 과정과 제어 흐름 보호가 만나는
부분을 더 공부해 보고 싶어졌다. 공격 관점에서는 Signal Handling 외에
다른 예외 처리 과정에서도 비슷한 우회 가능성이 있는지 분석할 수 있을 것이고
방어 관점에서는 커널이 사용자 공간의 실행 상태를 복원하기 전에 어떤
검증을 해야 하는지 생각하면 대응책을 만들 수 있을 것이다.

## 참고 자료

1.  Marcos Bajo, Ritvik Goyal, Apostolos Chatzianagnostou, and Christian
    Rossow, "Crashing Through Defenses: Exploiting Segfaults and
    Chaining around Intel CET," IEEE Symposium on Security and
    Privacy, 2026. [논문
    PDF](https://syssec.cispa.de/papers/sfop-ieeesp2026.pdf)
2.  [SFOP 공식 GitHub 저장소](https://github.com/signal-sfop/sfop)
