---
name: opt_kernel 메인 어젠다
description: opt_kernel 프로젝트 핵심 방향성 — Triton 수준 GEMM을 CUDA C++로 직접 구현 + Triton 불가 기능 추가
type: project
---

## 메인 어젠다: CUDA C++ Native High-Performance GEMM (2026-03-15 채택, 2026-03-18 재확인)

Triton GEMM이 자동화하는 기법들을 CUDA C++로 직접 구현하되, Triton으로 불가능한 저수준 최적화를 추가하여 성능 우위 확보.

### ★ 2026-03-18 현재 우선 순위 변경 ★

**이전 우선 순위:**
1. Tier 3 raw launcher (ATen 오버헤드 제거)
2. 커스텀 양자화 GEMM/GEMM

**변경 후 우선 순위:**
1. ★★★ **llama.cpp 백엔드 통합** (K30 Phase 1)
   - Qwen3.5-0.8B 로드 테스트
   - 208 tok/s 성능 확인
   - 커스텀 포맷과의 공존 방식 확립

2. **커스텀 양자화 GEMM/GEMM**
   - llama.cpp에 없는 60+ 포맷
   - Grid VQ, Learned LUT, NVFP4, INT8 Delta 등

3. **Tier 3 raw launcher** (보류, K31)
   - Low-level 90% 완료
   - Integration 문제로 일시 중단
   - Phase 2에서 재개

### 핵심 원칙

1. **Triton이 하는 것을 CUDA C++로 직접 구현**
   - 타일링 (블록 단위 행렬 분할)
   - Shared memory 스테이징 (global → smem → register)
   - 소프트웨어 파이프라이닝 (메모리 로드 + 연산 오버랩)
   - 워프 스케줄링

2. **Triton으로 불가능한 기능 추가** (차별화 포인트)
   - Warp-level intrinsics (`__shfl_sync`, `__ballot_sync`)
   - Tensor Core 직접 제어 (`mma.sync`, `wgmma` via CUTLASS/CuTe)
   - 비동기 복사 (`cp.async`, TMA — Hopper/Blackwell)
   - 커스텀 메모리 레이아웃 (swizzle, bank conflict 회피)
   - Cross-CTA 통신 (cooperative groups, cluster launch)
   - 인라인 PTX (명령어 수준 제어)

### 기존 K27 Integer-Domain Compute와의 관계

K27 dp4a/IMMA는 이 어젠다의 첫 번째 구현 사례. 앞으로의 모든 커널 개발은 이 어젠다를 기준으로 설계.

**Why:** Triton은 빠른 프로토타이핑에 유리하지만, 세밀한 하드웨어 제어가 불가능. CUDA C++로 동일 기법을 직접 구현하면서 저수준 최적화를 추가하면 Triton 대비 성능 우위 + 커스텀 양자화 GEMM에 최적.
**How to apply:** 새 커널 설계 시 항상 (1) Triton 수준 기법 포함 여부 (2) Triton 불가 최적화 적용 가능 여부를 점검. CUTLASS/CuTe 패턴 참조.

### 2026-03-18 변경 사유

**반복되는 문제 패턴 발견:**
- K29: Tier 3 시도 → segfault → 전략 전환
- K31: Low-level 작동 ✅, Integration 실패 ❌

**종합 진단 결론:**
- 실제 작동하는 것 (llama.cpp)부터 확보
- K31 Tier 3 low-level 작업들은 Phase 2에서 재활용
- 즉시 성능 확보가 중요 (208 tok/s vs 32 tok/s)
