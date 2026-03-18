---
name: 3090-build-limit
description: 3090 LXC 빌드 시 코어 제한 설정
type: feedback
---

3090 LXC에서 CUDA 빌드 시 `MAX_JOBS=16`로 제한해야 함.

**Why:** 3090 LXC는 CPU/메모리 리소스가 제한적이어서 무제한 병렬 빌드 시 CPU와 메모리가 100% 점유되어 시스템이 멈춤.

**How to apply:**
```bash
# 3090에서 opt_kernel 빌드 시
MAX_JOBS=16 .venv/bin/pip install -e . --no-build-isolation
```
