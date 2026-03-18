---
name: opt_kernel + IE 통합 실행 계획서 (v3 최종)
description: 68건 통합 (opt_kernel 29 + IE 35 + 인프라 3 + import 검증 1). 2차 리뷰 운영 공백 반영.
type: project
---

## 통합 현황

| 프로젝트 | 버전 | MVP | 이슈 | CRITICAL |
|---------|------|-----|------|---------|
| opt_kernel | v0.9.9 | 82% | 29건 | 5건 |
| inference-engine | v0.1.0 | 67% | 35건 | 4건 |
| 인프라 (P12, P16-P18) | - | - | 4건 | 0건 |
| **합계** | | | **68건** | **9건** |

---

## Sprint 1: CRITICAL + 즉시 가능 (1주, 전부 병렬)

```
┌──── opt_kernel (6건) ──────────┐  ┌──── IE (4건) ────────────────┐
│ N1: cudaMalloc 에러 체크        │  │ N12: SSE JSON double-encoding│
│ N2/N43: softmax div-by-zero    │  │ N13: generate() lock 레이스  │
│ N3+N26+N27+N10: graph_pool.cpp │  │ N14: Hot reload OOM          │
│   (4건 통합 1PR)               │  │ N36: MoE scatter_add_ 차원   │
│                                │  │                              │
│ 예상: 3-5일                    │  │ 예상: 2-3일                  │
└────────────────────────────────┘  └──────────────────────────────┘

┌──── 인프라 (1건, 독립) ─────────┐  ┌──── Anytime Pool (4건) ──────┐
│ P12: ConnectX IP netplan 영구화 │  │ N8:  shape 검증 [opt_kernel] │
│ (30분, 즉시 가능)               │  │ N18: path traversal [IE]     │
│                                │  │ N20: GGUF config 검증 [IE]   │
└────────────────────────────────┘  │ N33: dangling pointer [opt]  │
                                    │ (언제든 틈나면 진행)          │
                                    └──────────────────────────────┘
```

변경: N10을 Sprint 7→1로 당김 (graph_pool.cpp 통합 수정), P12를 Sprint 4→1로 당김.

---

## Sprint 2: MVP 블로커 (2주, 트랙 A/B 병렬)

```
트랙 A: opt_kernel
  P4/N5+N9: binding.cpp 검증 + layer_offsets 경계 (2-3일)
    │  (N9를 Sprint 6→2로 당김, C++ 검증 통합)
    ▼
  P1: Tier 3 raw launcher 통합 (1-2주)
    │  ⚠️ 2주 하드 데드라인. 미완 시 Tier 2 default + Tier 3 experimental.
    │  ★ IE forward_cpp.py 업데이트 포함 (크로스 의존성)
    ▼
  opt_kernel import 검증: IE의 from opt_kernels import 경로 확인 (0.5일)

트랙 B: IE
  P5: KV Transport 통합 (3-5일) ─┐
                                  ├→ P3: 분산 E2E 테스트 (2-3일)
  P2: Gateway→Coordinator (2-3일) ┘
      (P2, P5 병렬 → P3 순차)
```

변경: N9를 Sprint 6→2로 당김. P1에 IE forward_cpp.py 크로스 의존성 명시. import 검증 태스크 추가.

### 의존성 그래프

```
Sprint 1 완료
  ├→ P4/N5/N9 (C++ 검증) → P1 (Tier 3) → import 검증
  │                              │
  │                        opt_kernel v1.0.0-rc
  │
  ├→ P5 (KV 통합) ──┐
  │                   ├→ P3 (E2E 테스트) → IE v0.2.0-rc
  └→ P2 (Gateway)───┘
```

### ★ Sprint 2 후 결정 게이트

> "분산 decode를 opt_kernel Tier 3로 전환할 것인가, llama-server를 유지할 것인가?"
> - P1 성공 (200+ tok/s): Tier 3 decode 경로 추가 검토 → Sprint 4 우선순위 변경
> - P1 미완/부분: llama-server 유지, P14 (C 서버) 우선

---

## Sprint 3: 안정성/정확성 (2주, 19건, 병렬)

```
┌──── opt_kernel (8건) ──────────┐  ┌──── IE (10건) ────────────────┐
│                                │  │                                │
│ 즉시 가능 (P1과 무관):         │  │ 전부 독립:                     │
│ ├ N4+N35: fused_model_forward  │  │ ├ P9: ITL 메트릭              │
│ │   (ptr packing + contiguous, │  │ ├ N16: cat 텐서 메모리 누수    │
│ │    통합 1PR)                 │  │ ├ N17: quant fallback 경고     │
│ ├ N6: smem 크기 제한           │  │ ├ N22: tokenizer KV 정리       │
│ ├ N7: async memcpy 동기화      │  │ ├ N23: cache reset 메모리      │
│ └ N44: log(0) 방지             │  │ ├ N38: attention mask bf16     │
│                                │  │ └ N15: KV cache off-by-one     │
│ P1 80% 후 진행:               │  │                                │
│ ├ P6: K27b IMMA               │  │ 샘플러 통합 1PR:              │
│ ├ P7: K27c INT4 TC            │  │ ├ N37: Mirostat batch mu       │
│ ├ P8: codebook 오버플로우      │  │ ├ N39: Mirostat .squeeze()     │
│ └ P10: head_dim 정렬           │  │ └ N42: Top-p off-by-one       │
│                                │  │                                │
│ 예상: 2-3주                    │  │ 예상: 2주                      │
└────────────────────────────────┘  └────────────────────────────────┘
```

변경: N35를 Sprint 6→3으로 당김 (N4와 fused_model_forward 통합). N37+N39+N42 통합 1PR. P6/P7/P8은 P1 80% 후로 지연.

---

## Sprint 4: 분산/네트워크 (1-2주, 4건)

Sprint 2 P2/P3/P5 완료 후 진행. P12는 Sprint 1로 이동됨.

```
P11: Worker 장애 복구 ──→ P15: Circuit breaker

P13: 세션 정리→Worker 통지 ──→ P14: KV Transport C 서버
                                   ★ P5 설계 기반 (P5→P14 의존성)
                                   ★ RDMA-ready 소켓 아키텍처
```

---

## Sprint 5: 하드웨어 (물리 작업)

```
P16: GPU SLOT6 이동 → P18: Resizable BAR → P17: ConnectX-5 교체
P19: cp.async 정리 (독립)
P20: bank conflict 분석 (독립)
```

★ 하드웨어 지연 시 v1.0.0/v0.2.0은 하드웨어 없이 릴리스. P16-P18은 v1.1.0 타겟.

---

## Sprint 6: 보안/검증 (1주, 병렬)

```
opt_kernel (4건):          IE (4건):
├ N9: (Sprint 2로 이동됨)   ├ N18: (Anytime pool)
├ N29: Marlin multi-GPU     ├ N19: tool call JSON
├ N31: RoPE head_dim=1      ├ N40: KV graph batch
├ N33: (Anytime pool)       └ N41: head_dim 추론
└ N35: (Sprint 3으로 이동됨)
```

잔여: opt_kernel 2건 (N29, N31) + IE 3건 (N19, N40, N41) = **5건**

---

## Sprint 7: 문서/테스트 (1주, 병렬)

```
opt_kernel (5건):           IE (8건):
├ P21: 양자화 가이드         ├ P22: 분산 런북
├ P25: Marlin 정리           ├ P23: Request ID
├ P26: static_assert         ├ P24: XML edge case
├ P27: CUDA arch 검증        ├ P28: 로드 테스트
└ N11: underflow 통일        ├ N21: client disconnect
                             ├ N24: format 조기 검증
                             └ N45+N46: subprocess 통합 1PR
```

---

## 최적화된 타임라인

```
Week 1:    Sprint 1 (10건 + Anytime 4건)
Week 1-3:  Sprint 2 (P4/N5/N9 → P1 시작, P2+P5 병렬)
Week 2-4:  Sprint 3 시작 (IE 전부, opt_kernel 안전 항목만)
Week 3-4:  Sprint 2 P1 완료 → 결정 게이트 → Sprint 3 opt_kernel 나머지
Week 4-5:  Sprint 4 (분산 강화)
Week 5-7:  Sprint 5 (하드웨어) + Sprint 6 (보안) 병렬
Week 7-8:  Sprint 7 (문서/테스트)

Week 8:    opt_kernel v1.0.0 + IE v0.2.0 릴리스
```

## 수정된 통계

| Sprint | 건수 | 변경 사항 |
|--------|------|----------|
| 1 | 11+4 | N10, P12 당김, Anytime pool 추가 |
| 2 | 6 | N9 당김, import 검증 추가 |
| 3 | 18 | N35 당김, P6/P7/P8 P1 후 지연 |
| 4 | 5 | P14 RDMA-ready 요구 |
| 5 | 5 | 변경 없음 |
| 6 | 5 | N9/N33/N35 이동으로 축소 |
| 7 | 13 | N45+N46 통합 |

## Descope 후보 (MVP에 불필요한 항목)

긴급 시 제외 가능 (2-3주 절약):
- P6 (K27b): 이미 비활성, 문서화만
- P7 (K27c): 이미 xfail, 문서화만
- P9 (ITL 메트릭): 관측성, 기능 아님
- P20 (bank conflict): 분석, 수정 아님
- P22 (런북): 문서
- P25 (Marlin 정리): 정리
- P28 (로드 테스트): 테스트

## 크리티컬 패스

```
Sprint 1 (3일) → P4/N5/N9 (2일) → P1 (2주) → K27 검증 (1주) → v1.0.0
Sprint 1 (2일) → P2+P5 (5일) → P3 (3일) → P11 (1주) → v0.2.0
```

병목: **P1 Tier 3 (2주)**. 완화: Tier 2 fallback 전략.

---

## 운영 규칙

### Sprint 종료 기준
- **Sprint 1**: 모든 CRITICAL (9건) 해결 + 회귀 테스트 통과 → Sprint 2 시작
- **Sprint 2**: P1 완료 or 2주 데드라인 + 결정 게이트 + 회귀 테스트
- **Sprint 3**: opt_kernel K27 검증 + IE 샘플러 검증 + 회귀 테스트
- **Sprint 4-7**: 각 Sprint 항목 80% 이상 완료 시 다음 진행 (잔여는 이월)

### Anytime Pool 규칙
- N8, N18, N20, N33: 릴리스 전 완료 필수 (Sprint 7 종료까지)
- 각 Sprint 조기 완료 시 Anytime에서 선택
- 개별 30-60분 작업, 별도 추적

### 테스트 전략
- Sprint 1 후: 전체 테스트 스위트 실행 (CRITICAL 수정 회귀 확인)
- Sprint 2 후: opt_kernel 빌드 + IE E2E 테스트
- Sprint 4 후: 분산 E2E 테스트 (P3 작성분)
- Sprint 7 후: 최종 회귀 + P28 로드 테스트
- **N13 수정 후**: generate() 스루풋 벤치마크 (회귀 확인)
- **N36 수정 후**: MoE 하류 호출자 검증 (API 변경 여부)

### 릴리스 기준
- **opt_kernel v1.0.0**: Phase 0 CRITICAL 0건, 테스트 통과, Tier 3 200+ tok/s (or Tier 2 fallback)
- **IE v0.2.0**: Phase 0 CRITICAL 0건, E2E 테스트 통과, 분산 dispatch 동작
- Anytime pool 4건 완료
- Descope 후보 항목은 릴리스 노트에 "Known Limitations"로 명시

### 수정된 통계

| Sprint | 건수 |
|--------|------|
| 1 | 11 + Anytime 4 |
| 2 | 7 (import 검증 포함) |
| 3 | 19 |
| 4 | 4 |
| 5 | 5 |
| 6 | 5 |
| 7 | 13 |
| **합계** | **68** (64 이슈 + 인프라 4) |

**Why:** v3 최종. 2차 리뷰 운영 공백 8건 반영 (종료 기준, 테스트, 릴리스 기준).
**How to apply:** Sprint 1부터. 종료 기준 충족 후 다음 Sprint. Anytime은 릴리스 전 필수 완료.

---

## K29 Tier 3 진행 상황 (2026-03-18)

### 완료 작업
- ✅ Mamba weight FP32→FP16 변환 (`_ensure_fp16` 함수 강화)
- ✅ sym_decode_step 4D recurrent_state 지원
- ✅ Mamba 분기 구현 확인 (`make_hybrid_layer_fn`)
- ✅ CUDA Graph capture 성공 (1.13x speedup)
- ✅ Tier 2 vs Tier 3 benchmark

### 결과 (GB10)
| 항목 | Tier 2 (C++ ATen) | Tier 3 (Raw Kernels) | Speedup |
|------|-------------------|---------------------|---------|
| 처리량 | 157.1 tok/s | 151.9 tok/s | 0.97x |
| 시간 | 6.366 ms/token | 6.585 ms/token | - |
| CUDA Graph | - | 1.13x | ✓ |

### 다음 단계
1. **RTX 3090 재벤치마크**: GB10에서는 0.97x로 느리지만, RTX 3090에서는 1.5-2x expected
2. **Mamba kernel 최적화**: FP32 SSM 연산으로 정확도 향상 (llama.cpp 호환)
3. **긴 시퀀스 테스트**: 512+ token에서 성능 확인

### P1 Tier 3 상태
- **진행률**: ~70% 완료
- **완료**: C++ 구현, Hybrid dispatch, SSM state management
- **남음**: 성능 최적화 (RTX 3090 검증 필요)

