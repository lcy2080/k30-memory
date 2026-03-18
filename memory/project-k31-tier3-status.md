---
name: K31 Tier 3 실제 상태 정리
description: K31 Tier 3 구현 현황 - 문서와 실제 불일치, 남은 작업 명세화
type: project
---

## K31 Tier 3 실제 상태 (2026-03-18 기준)

### 문서 vs 실제 불일치

문서(`Docs/K31_tier3_raw_launcher_plan.md`) 상태:
```
| Mamba Tier 3 | ❌ | make_mamba_layer_fn 미구현 |
| Tier 3 init 성공 | ❌ | FP16 weight 변환 완료했으나 segfault |
```

실제 코드 상태:
```
| make_hybrid_layer_fn | ✅ 존재 (line 1700-1809) |
| MambaLayerPtrs | ✅ 존재 (decode_state.h) |
| CUDA 커널 3종 | ✅ 존재 (fused_gated_delta_rule_kernel.cu) |
| sym_decode_step dispatch | ✅ 존재 (line 2025-2073) |
```

### 실제 완료된 작업 (이번 세션)

1. **Mamba SSM State Memory Alignment Fix**
   - `python/aligned_tensor.py` - 128-byte aligned allocation
   - 테스트 통과 (head_dim=128, aligned dimensions only)

2. **Tier 3 "프레임워크" 검증**
   - 개별 커넬들은 작동
   - 하지만 전체 시스템 init 실패

### 실제 남은 문제들

#### 1. Weight Packing/Init 실패
```
RuntimeError: expected scalar type Half but found Char
```
- weight가 제대로 pack되지 않음
- `IE Qwen35 packing`이 작동하지 않을 수 있음

#### 2. 전체 시스템 통합 테스트 부족
- 개별 커널 테스트: ✅
- 전체 모델 init: ❌
- 실제 Qwen3.5 모델 로드: ❌

#### 3. Non-aligned dimensions 지원
- head_dim=170 등에서 여전히 실패
- aligned allocation으로 해결 시도

## 다음 단계 (정리)

### 즉시 필요한 것
1. **Weight packing 디버깅** - 전체 init이 작동하도록
2. **전체 모델 테스트** - 실제 Qwen3.5-0.8B 로드

### 선택사항
1. 문서 업데이트 - "현재 상태" 섹션을 실제와 일치하도록
2. Non-aligned dimension 해결 방법
3. Phase C, D, E 진행 (혼란 방지)

## 결론

**K31 Tier 3 "프레임워크"는 90% 구현되어 있으나,**
- **integration 문제**로 전체 시스템이 작동하지 않음
- **정렬 문제**로 특정 dimensions에서 실패

**Why:** 이번 세션에서는 정렬 문제만 해결했고, integration은 건드리지 않음

**How to apply:** 전체 시스템이 작동하도록 디버깅한 후, Phase C/D/E 진행