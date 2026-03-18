---
name: K30 Phase 1-1 결과
description: llama.cpp Qwen3.5 지원 확인 결과 - 2026-03-18
type: project
---

## K30 Phase 1-1: llama.cpp Qwen3.5 지원 확인 완료

### 확인 결과: ✅ 완전 지원

llama.cpp는 Qwen3.5 (Mamba + Attention Hybrid)를 완전히 지원합니다.

### 1. C++ 백엔드 구현

**아키텍처 정의** (`src/llama-arch.h`):
```cpp
LLM_ARCH_QWEN35,      // Qwen3.5
LLM_ARCH_QWEN35MOE,   // Qwen3.5 MoE
```

**모델 구현** (`src/models/qwen35.cpp`):
- `llm_build_qwen35` - Qwen3.5 hybrid model builder
- `is_recurrent(il)` - Mamba vs Attention layer detection
- `build_layer_attn_linear` - Linear attention (Mamba) layer
- `build_layer_attn` - Standard attention layer

**Mamba 기반 구현** (`src/models/mamba-base.cpp`):
- SSM convolution states
- Recurrent states
- `ggml_ssm_conv` operation

### 2. Python 변환 스크립트

**모델 클래스 등록** (`convert_hf_to_gguf.py`):
```python
@ModelBase.register("Qwen3_5ForConditionalGeneration", "Qwen3_5ForCausalLM")
class Qwen3_5TextModel(_LinearAttentionVReorderBase):
    model_arch = gguf.MODEL_ARCH.QWEN35
```

**Linear Attention 지원**:
- `_LinearAttentionVReorderBase` extends `Qwen3NextModel`
- V head reordering for ggml broadcast optimization
- Mamba SSM 파라미터 처리:
  - `in_proj_qkv` - QKV projection
  - `in_proj_z` - Z gate
  - `in_proj_b/a` - Beta/Alpha
  - `A_log`, `dt_bias`, `dt_proj` - SSM parameters
  - `conv1d` - Convolution
  - `out_proj` - Output projection

**Tokenizer 지원**:
- `_set_vocab_qwen()` - Qwen tokenizer 처리
- `get_vocab_base_pre()` returns "qwen35" for Qwen3.5-9B-Instruct

### 3. Hybrid 아키텍처 지원

Qwen3.5는 Mamba + Attention hybrid를 지원합니다:

```
for (int il = 0; il < n_layer; ++il) {
    if (hparams.is_recurrent(il)) {
        // Linear attention layer (Mamba)
        cur = build_layer_attn_linear(inp->get_recr(), cur, il);
    } else {
        // Full attention layer
        cur = build_layer_attn(inp->get_attn(), cur, inp_pos, sections, il);
    }
}
```

### 4. Qwen3.5-0.8B 호환성

Qwen3.5-0.8B 아키텍처:
- head_dim = 384 (128-byte aligned ✅)
- 24 layers: 18 Mamba + 6 Attention
- vocab_size = 151936

llama.cpp는 이 모든 특성을 지원합니다.

### Phase 1-1 완료 체크리스트

- [x] llama.cpp 클론 완료 (`/tmp/llama.cpp`)
- [x] Qwen3.5 아키텍처 지원 확인
- [x] Mamba layer 지원 확인
- [x] 변환 스크립트 호환성 확인
- [x] Hybrid 모델 지원 확인

### 다음 단계: Phase 1-2

**Qwen3.5-0.8B GGUF 변환**

1. Hugging Face에서 Qwen3.5-0.8B 다운로드
2. `convert_hf_to_gguf.py`로 GGUF 변환
3. 양자화 (Q4_K_M, Q5_K_M, Q6_K, Q8_K)
4. llama.cpp 로드 테스트

**빌드 필요사항**:
```bash
cd /tmp/llama.cpp
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
```

**Why**: llama.cpp는 Qwen3.5-0.8B를 완전히 지원하므로 바로 변환 및 벤치마크 가능.
**How to apply**: Phase 1-2에서 `convert_hf_to_gguf.py`를 사용하여 Hugging Face 모델을 GGUF로 변환.
