---
name: IE Inference Engine 비전 및 어젠다
description: IE 프로젝트 핵심 방향성 — 기존 플랫폼 장점 통합 + 모델별 커스텀 최적화/양자화
type: project
---

## IE (Inference Engine) 비전 (2026-03-15 정립)

기존 추론 플랫폼(vLLM, TensorRT-LLM, Ollama, llama.cpp)의 각 장점을 통합하고,
모델별 커스텀 최적화 및 커스텀 양자화를 자유롭게 추가할 수 있는 research inference runtime.

### 각 플랫폼에서 가져오는 장점

| 플랫폼 | 채택 요소 | IE 위치 |
|--------|----------|---------|
| vLLM | Continuous batching, PagedAttention, CUDA Graph | scheduler/, optimization/ |
| TensorRT-LLM | Fused kernel dispatch, IMMA 활용, 양자화 파이프라인 | quantization/, executor/ |
| llama.cpp | C++ full model forward (zero Python overhead), K/I-Quant 포맷 | opt_kernel fused_model_forward.cpp |
| Ollama | 사용 편의성, 모델 관리, 서빙 인터페이스 | serving/ (HTTP/gRPC) |

### 차별화 (기존 플랫폼이 못하는 것)

1. **모델별 커스텀 양자화**: 60+ 포맷, 레이어별 다른 전략 (YAML 프로파일)
2. **커스텀 CUDA 커널** (opt_kernel): Triton 수준 + 저수준 최적화 (메인 어젠다)
3. **이질적 하드웨어 클러스터**: GB10 prefill + RTX 3090 decode, 레이어/expert 단위 배치
4. **완전한 가시성/제어**: 모든 서브시스템 메트릭, prefill/decode 명시 분리

### 아키텍처 스택

```
사용자 → IE Serving (HTTP/gRPC)
       → Scheduler (continuous batching)
       → Executor (C++ model forward / CUDA Graph)
       → opt_kernels (CUDA C++ 커널)
       → Hardware (GB10 / RTX 3090 클러스터)
```

### 리포지토리

- IE: ~/ai/practice/inference-engine (git.dan-lab.dev/ai/engine/practice)
- opt_kernel: ~/ai/opt_kernel (git.dan-lab.dev/ai/engine/opt_kernel)

### 서빙 레이어 로드맵 (2026-03-16 추가)

- **Tool Calling 호환성**: 모델별 tool call 포맷 자동 감지/변환. **IE 서빙 레이어에서 플러그인 구조로 구현** (proxy에서 다 커버하지 않음).
  - OpenAI `tool_calls` 필드 (표준), Nemotron/Qwen3 XML `<tool_call>` 태그, Hermes JSON 등
  - 파라미터 검증/정규화도 IE 레벨에서 처리
- **Reasoning/Thinking 호환성**: 모델별 reasoning 출력 방식을 IE에서 파서 플러그인으로 처리.
  - `reasoning_content` 필드, `<think>` 인라인 태그 등
- **API 호환성 확장**: 현재 Anthropic→OpenAI 변환 구현 완료. 역방향 및 Gemini API 검토 예정.
- **현재 anthropic_proxy 역할**: 기본적인 API 포맷 변환만 담당. 세부 파싱/검증은 IE 본체로 이관 예정.
- **배경**: Nemotron Nano(활성 3B)는 tool call 포맷 생성 불안정 → Dense 모델(Qwen3.5 27B 등) 권장.

**Why:** 기존 플랫폼은 각각 장점이 있지만 커스텀 양자화, 모델별 최적화, 이질 하드웨어 지원에 한계. IE는 이 모든 것을 하나의 모듈러 플랫폼에서 실험 가능.
**How to apply:** IE 관련 작업 시 항상 (1) 어떤 플랫폼의 장점을 구현하는지 (2) 어떤 차별화를 추가하는지 점검.
