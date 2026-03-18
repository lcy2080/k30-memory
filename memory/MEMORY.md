# 메모리 인덱스

## 사용자
- [user-profile.md](user-profile.md) — 사용자 프로필 및 선호

## 환경
- [env-setup.md](env-setup.md) — 시스템 환경 및 설치된 주요 패키지
- [cluster-setup.md](cluster-setup.md) — AI 클러스터 네트워킹 (GB10 ↔ 3090) 설정 및 이슈

## 피드백
- [feedback-3090-build.md](feedback-3090-build.md) — 3090 LXC 빌드 시 MAX_JOBS=16 제한 필수

## 프로젝트
- [project-ie-vision.md](project-ie-vision.md) — IE 비전: 기존 플랫폼(vLLM/TRT-LLM/llama.cpp/Ollama) 장점 통합 + 커스텀 최적화
- [project-opt-kernel-agenda.md](project-opt-kernel-agenda.md) — opt_kernel 메인 어젠다: Triton GEMM을 CUDA C++로 직접 구현 + 저수준 최적화
- [project-k27-status.md](project-k27-status.md) — K27 Integer-Domain Compute 구현 상태, 벤치마크, 남은 작업 (K28a/K27b IMMA)
- [project-k29-qwen35.md](project-k29-qwen35.md) — K29 Qwen3.5 Tier 3 실험 결과 (Tier2=Python, Tier3 segfault → 전략 전환)
- [project-k30-strategy.md](project-k30-strategy.md) — ★현재★ K30 전략: llama.cpp K-quant 백엔드 + opt_kernel 커스텀 포맷 집중
- [project-k31-tier3-status.md](project-k31-tier3-status.md) — K31 Tier 3 구현 현황 - 문서와 실제 불일치, 남은 작업 명세화
- [project-k31-tier3-diagnosis-20260318.md](project-k31-tier3-diagnosis-20260318.md) — K31 Tier 3 정밀 진단 결과 (2026-03-18) - 작동 기능, 알려진 문제, 권장 다음 단계
- [project-k30-phase1-status.md](project-k30-phase1-status.md) — ★현재★ K30 Phase 1 llama.cpp 백엔드 통합 진행 상태
- [k30-phase1-1-findings.md](k30-phase1-1-findings.md) — K30 Phase 1-1 결과: llama.cpp Qwen3.5 지원 확인 완료
- [k30-phase1-2-findings.md](k30-phase1-2-findings.md) — K30 Phase 1-2/1-3 결과: Qwen3.5-0.8B GGUF 변환 및 193.5 tok/s 달성
