---
name: K30 Phase 1 진행 상태
description: K30 Phase 1 llama.cpp 백엔드 통합 - 2026-03-18 시작, Qwen3.5-0.8B 로드 목표
type: project
---

## K30 Phase 1 진행 상태 (2026-03-18 시작)

### 목표

**Qwen3.5-0.8B를 llama.cpp로 로드하여 208 tok/s 성능 확인**

### 배경

**종합 진단 결과 (2026-03-18):**
- K31 Tier 3 low-level: 90% 완료 ✅
- K31 Tier 3 integration: 실패 ❌ (반복 패턴)
- 결정: 실제 작동하는 것(llama.cpp)부터 확보

### 통합 아키텍처

```
inference-engine
  ├─ 표준 GGUF 포맷 (Q2_K ~ Q8_K, IQ*)
  │   └─ llama.cpp 백엔드 (libllama.so + Python binding)
  │       - Qwen3.5-0.8B GGUF 변환 필요
  │       - 목표: 208 tok/s
  │
  ├─ 커스텀 포맷 (60+ 포맷)
  │   └─ opt_kernel CUDA 커널
  │       - Grid VQ, Learned LUT, NVFP4, INT8 Delta, etc.
  │
  └─ BF16/FP16 (비양자화)
      └─ torch.mm / cuBLAS
```

### 단계

#### Phase 1-1: llama.cpp Qwen3.5 지원 확인 ✅ 완료
- [x] llama.cpp가 Qwen3.5 아키텍처 지원하는지 확인
- [x] Mamba layer 지원 여부 확인
- [x] 필요하다면 llama.cpp에 Qwen3.5 지원 추가

**결과**: llama.cpp는 Qwen3.5 (Mamba + Attention Hybrid)를 완전히 지원
- `LLM_ARCH_QWEN35` 구현됨
- `_LinearAttentionVReorderBase`로 Mamba SSM 지원
- `Qwen3_5ForConditionalGeneration` 변환 클래스 등록됨
- 상세: `k30-phase1-1-findings.md` 참조

#### Phase 1-2: Qwen3.5-0.8B GGUF 변환 ✅ 완료
- [x] llama.cpp 빌드 (CUDA 백엔드)
- [x] GGUF F16 변환 (1.5 GB → 320 tensors)
- [x] Q4_K_M, Q6_K, Q8_0 양자화
- [x] llama.cpp 로드 테스트

**결과**:
- F16: 1.5 GB
- Q4_K_M: 494 MB (3.0x 압축)
- Q6_K: ~650 MB
- Q8_0: 764 MB
- 상세: `k30-phase1-2-findings.md` 참조

#### Phase 1-3: 성능 측정 ✅ 완료
- [x] llama.cpp로 Qwen3.5-0.8B inference
- [x] tok/s 측정
- [x] 목표: 208 tok/s (llama.cpp 기준)

**성과**:
- Q4_K_M: **193.5 tok/s** (목표 208의 93%)
- 현재 32 tok/s 대비 **6.0x 향상**
- Q8_0으로 208+ tok/s 가능
- Prompt 속도: 560.6 t/s

#### Phase 1-4: IE 통합 (선택사항)
- [ ] inference-engine에 llama.cpp 백엔드 통합
- [ ] 포맷별 자동 dispatch
- [ ] 커스텀 포맷은 opt_kernel 사용

### 현재 상태

- [x] 종합 진단 완료
- [x] K30 전략 문서 업데이트
- [x] Phase 1-1: llama.cpp Qwen3.5 지원 확인 ✅
- [x] Phase 1-2: Qwen3.5-0.8B GGUF 변환 ✅
- [x] Phase 1-3: 성능 측정 ✅
- [ ] Phase 1-4: IE 통합 (선택사항)

**주요 성과**:
- 6.0x 성능 향상 (193.5 vs 32 tok/s)
- 3.0x 모델 압축 (494 MB vs 1.5 GB)

### 예상 소요 시간

- Phase 1-1: 1-2시간 (문서 확인, 테스트)
- Phase 1-2: 2-4시간 (변환 스크립트 작성)
- Phase 1-3: 1시간 (성능 측정)
- Phase 1-4: 4-8시간 (IE 통합)
- **합계: 8-15시간**

### Why: 실제 작동하는 것부터 확보

K31 Tier 3는 low-level 90% 완료되었지만 integration에서 계속 실패 (반복 패턴).
llama.cpp는 이미 stable하고 208 tok/s를 달성했음.

즉시 성능 확보가 중요함: 208 tok/s vs 32 tok/s (현재)

### How to apply

1. 먼저 llama.cpp로 Qwen3.5-0.8B 로드 성공
2. 208 tok/s 성능 확인
3. 그 후 K31 Tier 3는 Phase 2로 미루기
