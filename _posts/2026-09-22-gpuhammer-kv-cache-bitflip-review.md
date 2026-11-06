---
title: "[선행 연구 리뷰 & 후속 연구 방향성] GPUHammer & Bit-Flip Vulnerability of Shared KV-Cache Blocks"
date: 2026-09-22 21:11:29 +0900
categories: [논문/컨퍼런스]
tags: [paper-review, rowhammer, gpuhammer, kv-cache, radixmend]
---

우연히 cpu를 노린 Rowhammer 공격 논문을 접하고 나서 Rowhammer가 비트플립을 유도하는 과정에 매료되었다. 메모리는 주기적으로(refresh) 전하를 다시 채워주는데 **그 리프레시가 오기 전에 옆 행의 전하를 다 빼버릴 만큼 빠르게 두드리면** 값이 0에서 1로, 혹은 1에서 0으로 뒤집혀 버린다. 공격자는 남의 메모리를 읽은 적도 없다. 그저 자기 메모리를 반복해서 읽었을 뿐인데 그 물리적 부작용이 옆 행을 오염시킨다. 이런 호기심에서 '똑같이 DRAM이 들어가는 GPU에서도 동일한 방식으로 비트플립 유도가 가능하지 않을까?' 하는 생각이 들어 관련 논문을 찾아보았고 GPUHammer 논문을 발견하였다.

---

## 1. GPUHammer: Rowhammer Attacks on GPU Memories are Practical (USENIX Security 2025)

**Chris S. Lin, Joyce Qu, Gururaj Saileshwar (University of Toronto)** "GPUHammer: Rowhammer Attacks on GPU Memories are Practical"

### 배경: Rowhammer는 원래 CPU 문제였다

Rowhammer는 2014년  DDR3에서 처음 보인 DRAM 취약점이다. 메모리의 한 행(row)을 아주 빠르게 반복해서 읽으면 전기적 간섭 때문에 **물리적으로 인접한 행의 비트가 저절로 뒤집힌다.** 셀 간격이 촘촘할수록 심해진다. 이후 10년간 DDR4, DDR5로 확장되며 권한 상승·키 복구 등 실제 공격으로 발전했지만 줄곧 CPU에 연결된 메모리(DDR 계열) 문제로만 여겨졌다.

GPU 메모리(GDDR 계열)는 접근 패턴, 리프레시 주기, 물리적 구조가 달라서 "별개의 문제 영역"으로 취급됐다. 아무도 GPU에서 Rowhammer가 된다고 증명하지 못했다.

### 기여: GPU에서 처음으로 비트플립을 성공시켰다

GPUHammer는 **이산 NVIDIA GPU에서 최초로 Rowhammer 공격을 성공**시켰다.

구체적으로:

- **NVIDIA A6000(GDDR6)에서 4개 DRAM 뱅크에 걸쳐 최대 8개의 비트플립** 주입
- 이 비트플립으로 GPU 메모리에 상주하는 **DNN 가중치를 오염**시켜 ML 모델 정확도를 최대 80% 떨어뜨림 (ImageNet 정확도 80% → 0.1%)
- GDDR6의 방어 기법인 Target Row Refresh(TRR)가 있는데도 성공

이들이 넘어야 했던 GPU 특유의 난관 세 가지:

1. 물리 메모리→GDDR 뱅크/행 매핑이 비공개(proprietary)
2. 높은 메모리 지연 + 빠른 리프레시 주기 → hammering이 어려움
3. GDDR 메모리의 비공개 완화 기법 → FPGA 없이는 리버싱 곤란

해결한 방법:

(1) **비공개 매핑** → 드라이버의 결정론적 매핑을 리버싱하고 타이밍 사이드채널로 같은 뱅크 주소를 식별.

(2) **높은 지연/빠른 리프레시** → warp/스레드 병렬 hammering 커널 + PTX discard로 캐시 우회.

(3) **TRR 방어** → REF 명령과 동기화된 지연 삽입 + 17개 이상의 다중 공격 행으로 16개 용량의 TRR 샘플러를 압도.

이 논문은 비교적 최신이지만 그래도 1년 가량 시간이 흘렀기에 GPUHammer 관련 최신 연구 동향이 궁금해 찾아보니, LLM 서비스에 쓰이는 KV캐시를 오염시킬 수 있다는 최신 연구가 있었다.

---

## 2. Bit-Flip Vulnerability of Shared KV-Cache Blocks in LLM Serving Systems (arXiv 2604.17249)

**Yuji Yamamoto, Satoshi Matsuura** "Bit-Flip Vulnerability of Shared KV-Cache Blocks in LLM Serving Systems"

### 배경: 표적을 가중치에서 KV 캐시로 옮기다

이 논문은 GPUHammer의 연구를 이어받는다. 지금까지 비트플립 공격 연구(SBFA, GenBFA 등)는 전부 **정적인 모델 가중치**를 노렸다. 이 논문은 표적을 **동적으로 변하는 KV 캐시**로 옮긴다.

왜 KV 캐시가 위험한가? LLM 서빙 엔진은 속도를 위해 **prefix caching**을 쓴다. 여러 요청이 같은 앞부분(시스템 프롬프트 등)을 공유하면 그 부분의 계산 결과(KV 캐시)를 **하나의 물리 사본으로 공유**한다. 그런데 vLLM에는 이 공유 블록의 **무결성 보호가 전혀 없다.** 하드웨어나 OS가 이걸 강제하지 않아서 외부 비트플립에 무방비하다.

### KV 캐시 비트 플립의 세 가지 성질

소프트웨어 결함 주입(SFI)으로 BF16 KV 캐시의 비트를 XOR로 뒤집어 세 가지 성질을 밝혔다:

- **Silent divergence**: BF16 16비트 중 13개는 뒤집어도 문법적으로 정상이고 주제도 맞는, 그저 미묘하게 다른 출력을 낸다. 원본(clean baseline) 없이는 정상 응답과 구별 불가능. 나머지 3개(주로 지수 최상위 bit 14)는 NaN/inf로 출력이 붕괴(collapse)해서 오히려 탐지 쉬움.
- **Selective propagation**: 오염된 prefix를 공유하는 요청만 영향받는다.
- **Persistent accumulation**: 시간에 따른 감쇠가 없어서 이후 요청이 쌓일수록 피해가 선형으로 누적된다.

세 성질은 **조합**으로 작동한다. 조용한 변질 + 선택적 전파가 탐지를 회피하게 하고 그 상태로 지속적 누적이 견제 없이 진행돼 피해가 증폭된다.

실험 환경: **DGX Spark(128GB Unified Memory)**, 모델은 **Qwen3-8B**(메인)와 **DeepSeek-R1-Distill-Llama-8B**(교차검증), 데이터셋은 ShareGPT. 방법은 vLLM의 GPUModelRunner.execute\_model()을 직접 패칭해서 GPU 메모리 텐서에 XOR 주입.

### 결론

- **위협 규명:** LLM 서빙 시스템의 공유 Prefix KV-Cache 블록에 대한 단일 비트플립(Bit-Flip) 공격의 심각성을 최초로 체계적으로 분석하였다.
- **3가지 핵심 특성의 상호작용:** 2,400회의 오류 주입 실험과 24,000회의 후속 요청 분석을 통해 조용한 왜곡(Silent Divergence), 선택적 전파(Selective Propagation), 지속적 누적(Persistent Accumulation)의 3가지 요소가 결합되어 탐지를 우회하고 피해를 증폭시킨다는 점을 밝혔다. 이는 정적 모델 가중치 오염과는 완전히 다른 위협 양상이다.
- **대응책 검증:** 연구진이 제안한 체크섬 기반 대응책은 추가 오버헤드 없이 탐지 우회 및 피해 누적 고리를 끊어내고 누적 피해를 1개 배치 수준으로 제한함을 입증했다.

  ---

### **후속 연구 착안**

이 논문은 vLLM의 KV 캐시를 비트플립하는 공격의 영향과 그 대응책을 제시했다.

vLLM은 KV 캐시를 **블록 단위로 평면적으로(PagedAttention)** 저장한다. 그런데 또 다른 주요 엔진 SGLang은 **트리 구조(RadixAttention)**로 저장한다. 저장 구조가 근본적으로 다르면 위 세 가지 성질, 특히 피해 범위(blast radius)도 다르게 나타날 수 있다. 이를 바탕으로 SGLang에서 발생하는 비트플립에 대한 대응책도 설계하고자 한다.

**후속 논문의 방향성 :**

> ***SGLang의 공유 KV 캐시에서 발생하는 비트플립 오염에 대한 대응책으로,
> radix tree 구조를 고려한 무결성 검증 메커니즘을 제안한다.***
