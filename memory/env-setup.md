---
name: env-setup
description: 사용자 시스템 환경 — 하드웨어 및 주요 설정
type: user
---

- 하드웨어: NVIDIA GB10 (Grace Blackwell 기반 데스크탑/워크스테이션)
- OS: Linux 6.17.0-1008-nvidia (Ubuntu 24.04.4 LTS)
- 플랫폼: linux (aarch64), bash
- GPU 드라이버: 590.48.01, CUDA 13.1
- PyTorch venv: ~/ai/opt_kernel/.venv (2.10.0+cu128, NCCL 2.27.5)
- 시스템 NCCL: 2.29.7 (/usr/lib/aarch64-linux-gnu/libnccl.so.2)
- ConnectX-7: rocep1s0f0, enp1s0f0np0, 192.168.100.1/24, MTU 9000
