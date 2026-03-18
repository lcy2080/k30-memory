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
- [project-k29-status-20260318.md](project-k29-status-20260318.md) — K29 Tier 3 Mamba FP16 변환 완료 (2026-03-18) - 다음: sym_decode_step Mamba 분기
- [project-k30-strategy.md](project-k30-strategy.md) — ★현재★ K30 전략: llama.cpp K-quant 백엔드 + opt_kernel 커스텀 포맷 집중
- [project-k31-tier3-status.md](project-k31-tier3-status.md) — K31 Tier 3 구현 현황 - 문서와 실제 불일치, 남은 작업 명세화
- [project-k31-tier3-diagnosis-20260318.md](project-k31-tier3-diagnosis-20260318.md) — K31 Tier 3 정밀 진단 결과 (2026-03-18) - 작동 기능, 알려진 문제, 권장 다음 단계
- [project-k30-phase1-status.md](project-k30-phase1-status.md) — ★완료★ K30 Phase 1 llama.cpp 백엔드 통합 - 238.1 tok/s 달성
- [k30-phase1-1-findings.md](k30-phase1-1-findings.md) — K30 Phase 1-1 결과: llama.cpp Qwen3.5 지원 확인 완료
- [k30-phase1-2-findings.md](k30-phase1-2-findings.md) — K30 Phase 1-2/1-3 결과: Qwen3.5-0.8B GGUF 변환 및 193.5 tok/s 달성
- [k30-phase1-final.md](k30-phase1-final.md) — K30 Phase 1 최종 결과: 238.1 tok/s 달성, IE 통합 완료
- [k30-phase2-2-gridquant-lut-findings.md](k30-phase2-2-gridquant-lut-findings.md) — K30 Phase 2-2: Grid Quant & Learned LUT 테스트 결과 (Batch inference 5.37x)
- [k30-phase2-3-gridquant-limits.md](k30-phase2-3-gridquant-limits.md) — K30 Phase 2-3: GridQuant 파라미터별 한계 측정 (RVQ 10.48% error, 2.69x speedup)
- [k30-phase2-4-cdf-remap-dense.md](k30-phase2-4-cdf-remap-dense.md) — K30 Phase 2-4: CDF Remap Dense 모델 테스트 (9.53% error, 3.90x speedup, batch 최적)
- [k30-phase2-5-cdf-remap-7b-e2e.md](k30-phase2-5-cdf-remap-7b-e2e.md) — K30 Phase 2-5: CDF Remap Qwen2.5-7B E2E 테스트 (9.43% error, 2.48x speedup, 모든 크기에서 일관)
- [k30-phase2-6-quant-methods-comparison.md](k30-phase2-6-quant-methods-comparison.md) — K30 Phase 2-6: 양자화 방식 비교 (CDF Remap이 SmoothQuant보다 5.7%, llama.cpp보다 12.5% 더 낮은 error)
- [k30-phase3-1-ppl-wikitext.md](k30-phase3-1-ppl-wikitext.md) — K30 Phase 3-1: WikiText-2 PPL 측정 (CDF Remap이 FP16보다 4.1% 더 낮은 PPL 달성!)
- [k30-phase2-7-outlier-aware-findings.md](k30-phase2-7-outlier-aware-findings.md) — K30 Phase 2-7: Outlier-aware 양자화 실험 (7.25% error, 22.9% 개선)
- [project-3090-server-roadmap.md](project-3090-server-roadmap.md) — 3090 서버 AI 환경 구축 로드맵 (Proxmox→Ubuntu 마이그레이션 후)
- [project-migrate-proxmox-to-ubuntu-zfs-docker-k3s.md](project-migrate-proxmox-to-ubuntu-zfs-docker-k3s.md) — Proxmox→Ubuntu 마이그레이션 계획 (완료)
- [project-kv-transport-bench.md](project-kv-transport-bench.md) — KV Transport 벤치마크 결과 (Python TCP 0.25 GB/s, C 확장 필요)
- [mamba-dtype-analysis.md](mamba-dtype-analysis.md) — llama.cpp Mamba dtype 처리 분석: SSM은 FP32 전용, conv1d는 양자화 안 함 (2026-03-18)
- [project-mvp-roadmap-20260318.md](project-mvp-roadmap-20260318.md) — opt_kernel(29건) + IE(35건) MVP 이슈 목록
- [project-unified-plan-20260318.md](project-unified-plan-20260318.md) — ★현재★ 통합 실행 계획서 (64건, 7 Sprint, 8주, 의존성 분석)
