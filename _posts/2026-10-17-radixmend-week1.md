---
title: "[RadixMend 1주차] SGLang 코드 분석 및 실험 환경 세팅"
date: 2026-10-17 00:29:42 +0900
categories: [개발]
tags: [radixmend, sglang, kv-cache, triton]
---

> RadixMend 16주 프로젝트의 1주차 기록. 이번 주는 코드를 한 줄도 고치지 않고 SGLang이 KV 캐시를 실제로 어떻게 다루는지 따라가는 데 거의 다 썼다.

## 이번 주에 하기로 한 것 (WBS)

- SGLang 버전 확정, radix tree·KV 풀·attention 커널 코드 분석
- attention 백엔드 결정
- 워크로드 선정, 실험 환경·재현 스크립트 구성
- 결함 주입 도구 개발 착수

## 버전은 v0.5.22로 고정

분석을 시작한 시점의 최신 릴리스가 v0.5.22(commit baa9befa3)였고 프로젝트가 끝날 때까지 이 버전으로 간다. SGLang은 요즘 캐시 쪽 구조를 꽤 자주 바꾸고 있어서 중간에 버전을 올리면 패치를 처음부터 다시 맞춰야 할 수도 있다.

의존성을 보니 torch 2.14.1에 flashinfer 0.7.1(cu13)이다. CUDA 13 기준이라 GPU 서버를 받으면 드라이버가 CUDA 13을 지원하는지부터 봐야 한다.

## 코드를 읽어 보니 기술서랑 다른 게 꽤 있었다

기술서를 쓸 때는 mem\_cache/radix\_cache.py의 RadixCache를 머릿속에 두고 설계했다. 막상 요청 하나가 들어와서 KV가 저장되고 다시 읽히기까지를 따라가 보니 생각과 다른 부분이 여럿 나왔다.

- **lock\_ref가 노드에 바로 붙어 있지 않다.** 노드마다 FULL/SWA/MAMBA 같은 component별 데이터가 따로 있고 KV 슬롯 번호와 참조 카운트는 node.component\_data[FULL] 안에 들어 있다.
- **root는 KV를 안 가진다.** 이건 예상대로였다. 그래서 제일 많이 공유되는 노드는 root의 바로 아래 자식들, 즉 첫 레벨 노드다.
- **KV를 저장하는 커널은 Triton이 아니라 JIT로 빌드하는 CUDA C++ 코드다.** slot 번호가 범위 안인지 device assert로 확인하고 padding용 slot 0에는 쓰지 않는다. 저장할 때 체크워드를 같이 만들려면(encode-at-store) 이 CUDA 코드를 고치거나, 같은 자리에 Triton 커널을 끼워 넣어야 한다. 프로토타입은 Triton 쪽이 훨씬 빠를 것 같다.

보호할 인덱스 텐서들의 실제 모습은 이랬다.

| 텐서 | 위치 | dtype | 메모 |
|---|---|---|---|
| req_to_token | GPU | int32 | 0번 행은 padding |
| node.value | GPU | int64 | req_to_token 행을 복사해 둔 것 |
| 빈 슬롯 목록 (free_pages) | GPU | int64 | slot 0은 빠져 있음 |
| seq_lens | GPU + CPU 사본 | int64 | 여기저기서 산술에 씀 |
| out_cache_loc | GPU | int64 |   |

## attention 백엔드는 Triton으로

| 백엔드 | 언제 기본값인가 | 커널이 어디 있나 | 고칠 수 있나 |
|---|---|---|---|
| FlashInfer | A6000 같은 Ampere | 외부 라이브러리 | fork해야 해서 어렵다 |
| FlashAttention 3 | Hopper | sgl-kernel(CUTLASS) | 다시 빌드해야 해서 사실상 못 한다 |
| Triton | 직접 지정해야 함 | SGLang 저장소 안의 Python 코드 | 타일 읽은 직후에 바로 코드를 넣을 수 있다 |

고른 이유는 세 가지다.

1. decode, extend, kv\_indices 생성 커널이 전부 저장소 안에 Triton으로 있다.
2. decode 커널(\_fwd\_grouped\_kernel\_stage1)은 head\_dim이 128이면 K 타일의 열 하나가 딱 (토큰, KV 헤드) 벡터 하나다. 게다가 벡터 하나를 스텝·레이어마다 한 번만 읽는다. 읽자마자 검사하고 레지스터에서 고치면 되고 같은 벡터를 두 번 검사할 일도 없다.
3. extend 커널은 같은 prefix 벡터를 여러 번 읽는다. 오류 로그에 중복이 생긴다는 얘기라서 메모리 고치는 일을 별도 scrub 커널로 빼 둔 기술서 설계가 맞았다는 근거가 됐다.

A6000에서는 FlashInfer보다 느릴 수 있다. 그래서 비교 기준은 "아무것도 안 고친 Triton"으로 잡고 FlashInfer 수치는 참고용으로만 붙이기로 했다.

## 실험 환경과 워크로드

| 항목 | 값 |
|---|---|
| GPU | RTX A6000 48GB (GDDR6, 하드웨어 ECC 꺼짐) |
| 모델 | Qwen3-8B(주), Llama-3-8B-Instruct(보조), bf16 |
| 서버 옵션 | --attention-backend triton --page-size 1 --tp-size 1 --kv-canary none, --max-total-tokens 고정 |
| 샘플링 | greedy, seed 고정 (정답 출력이랑 비교해야 하니까) |

Qwen3-8B를 주 모델로 둔 건 선행 연구(Yamamoto & Matsuura, 2026)가 이 모델을 써서 결과를 바로 맞대 볼 수 있기 때문이다.

워크로드는 공유 prefix가 생기는 것 위주로 골랐다.

| 이름 | 구성 | 보고 싶은 것 |
|---|---|---|
| W1 | 공통 시스템 프롬프트 + ShareGPT 질문 | 첫 레벨 노드·sink가 망가졌을 때 피해 |
| W2 | GSM8K 8-shot, MMLU 5-shot | 정답률로 보는 피해 |
| W3 | 여러 턴 대화 | 파생 오염 |
| W4 | 가지가 여러 번 갈라지는 트리 | 자손 노드 수, 계보 기반 무효화 |
| W5 | W1 + SGLANG_RADIX_FORCE_MISS=1 | 공유를 끈 대조군 |

## 결함 주입 도구 시작

GPU가 없어도 테스트할 수 있게 torch 텐서만 건드리도록 만들었다. CPU에서 테스트하고 GPU에서는 같은 코드를 그대로 쓰면 된다.

```python
from radixmend import fault_inject as fi

# 3번 레이어 K 텐서, slot 1024, 2번 헤드, 77번 원소의 9번 비트를 뒤집는다
recs = fi.inject_kv(pool.k_buffer, pool.v_buffer,
                    [fi.KVFault(layer=3, kind="k", slot=1024, head=2, elem=77, bit=9)])
fi.revert(recs, k_buffer=pool.k_buffer, v_buffer=pool.v_buffer)  # 끝나면 되돌리기
```

트리에서 어디를 노릴지도 고를 수 있다. 첫 레벨 노드(first\_level\_nodes), sink 토큰 slot(sink\_slots), 많이 공유된 순서(nodes\_by\_sharing)를 지원한다.

## 트러블슈팅

### 1. 기본 radix tree가 Python이 아니었다

**문제.** 모듈 C는 트리 노드에 poisoned 플래그랑 epoch를 붙여야 한다. 그런데 기본 설정으로는 노드 객체를 아예 꺼낼 수가 없었다. 트리 어댑터 코드를 열어 보니 root\_node와 node\_by\_id가 그냥 NotImplementedError를 던지게 되어 있었다.

**원인.** v0.5.22의 기본 캐시는 UnifiedRadixCache이고 그 안의 트리 코어 기본값이 Rust 구현이다(SGLANG\_UNIFIED\_RADIX\_TREE\_CORE\_BACKEND = "rust"). Rust 코어는 Python 쪽에 노드를 넘겨주지 않는다. 예전 RadixCache는 이제 일부 경로에서만 쓰인다.

**해결.** 환경 변수로 Python 코어를 강제했다. 비교 실험이 공정하려면 기준선도 같은 설정으로 돌려야 한다. Python 코어가 얼마나 느린지는 GPU를 받으면 따로 재 볼 생각이다.

```bash
export SGLANG_UNIFIED_RADIX_TREE_CORE_BACKEND=python
```

### 2. 주입 도구가 SGLang의 실제 노드 구조를 못 읽었다

**문제.** 처음에는 주입 도구를 예전 TreeNode 기준으로 짜서 node.value, node.lock\_ref를 바로 읽게 했다. 그런데 기본 트리의 노드에는 그런 필드가 없었다.

**원인.** 위에서 말한 것처럼 지금 노드는 component\_data[FULL].value, component\_data[FULL].lock\_ref에 값을 들고 있다.

**해결.** 노드에서 값을 꺼내는 함수를 하나 두고 두 구조를 모두 읽게 바꿨다. 두 구조를 흉내 낸 가짜 노드로 테스트도 붙였다.

```python
FULL_COMPONENT = 0  # ComponentType.FULL

def node_value(node):
    if hasattr(node, "component_data"):
        return node.component_data[FULL_COMPONENT].value
    return node.value
```

### 3. int16 부호 비트를 뒤집는 값이 int16에 안 들어간다

**문제.** bf16 값을 int16으로 보고 15번 비트를 XOR하려면 1 << 15(=32768)를 넣어야 하는데, 이 값은 int16 범위(-32768~32767) 밖이다. int32 인덱스의 31번 비트도 마찬가지다. 부호 비트(15번)는 선행 연구에서 출력이 몰래 바뀐다고 보고된 위치라 꼭 뒤집을 수 있어야 했다.

**해결.** 맨 위 비트일 때는 같은 비트 패턴을 가진 음수로 바꿔서 XOR하도록 했다. int16이면 -32768, int32면 -2^31이다. 부호 비트를 뒤집으면 값의 부호만 바뀌는지 테스트로 확인했다.

```python
def _bit_value(bit, dtype):
    width = torch.iinfo(dtype).bits
    v = 1 << bit
    if torch.iinfo(dtype).min < 0 and bit == width - 1:
        v -= 1 << width   # 같은 비트 패턴의 음수
    return v
```

### 4. CPU용 PyTorch 설치가 막혔다

**문제.** 개발 환경에 GPU가 없어서 CPU 전용 PyTorch를 받으려고 download.pytorch.org/whl/cpu 인덱스를 썼더니, 네트워크 정책 때문에 403으로 막혔다.

**해결.** PyPI 기본 wheel을 그냥 받았다. CUDA 라이브러리가 같이 딸려와서 무겁긴 한데, GPU가 없으면 알아서 CPU로 돈다. 테스트를 돌리기엔 충분했다.

## 남은 일

- GPU 서버에서 드라이버 확인하고 Triton 백엔드로 모델 띄우기
- Python 트리 코어가 Rust보다 얼마나 느린지 재기
- 워크로드 생성과 서버 실행 스크립트는 GPU에서 처음 돌릴 때 마무리

## 다음 주

결함 주입기를 마무리하고 SECDED 참조 구현을 만들어서 단일·이중 오류를 하나도 빼지 않고 검증한다.
