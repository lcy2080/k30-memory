---
name: cluster-setup
description: AI 클러스터 네트워킹 최종 설정
type: project
---

## 하드웨어

### GB10 (DGX Spark)
- GPU: Blackwell (통합 메모리, 별도 VRAM 없음)
- NIC: ConnectX-7 (rocep1s0f0, enp1s0f0np0), 192.168.100.1/24, MTU 9000
- **GPUDirect RDMA 미지원** (통합 메모리 아키텍처, NVIDIA 공식 확인)

### 3090 서버 (Proxmox VE 9, 커널 6.17.13-2-pve)
- GPU: RTX 3090 — LXC 102 디바이스 바인드
- NIC: ConnectX-3 (SR-IOV 활성화), RoCE v1
- **GPUDirect RDMA 미지원** (GeForce 드라이버 차단)

## 현재 구성 — NCCL IB 0.90 GB/s (최종)

### 성능
- RDMA write GB10→3090: **38.32 Gbps**
- RDMA write 3090→GB10: **36.75 Gbps**
- RDMA read GB10←3090: **10.95 Gbps**
- NCCL IB all_reduce 256MB: **0.90 GB/s** ← 이 환경 최대
- GDR: 불필요 (GB10은 통합메모리=RDMA가 곧 GPU접근), RTX 3090측만 CPU복사 병목

### 구성 방식: SR-IOV VF(네트워크) + PF RDMA(IB)
- ConnectX-3 SR-IOV VF → LXC eth1 phys passthrough
- PF RDMA → /dev/infiniband rbind
- 영구 설정: mlx4-sriov.conf, hookscript, nic13 auto UP

### NCCL 환경변수
```
# LXC 102 (3090 측)
LD_PRELOAD=/home/nvwb/.local/lib/python3.12/site-packages/nvidia/nccl/lib/libnccl.so.2
NCCL_IB_HCA=rocep65s0 NCCL_IB_GID_INDEX=0 NCCL_IB_TIMEOUT=22 NCCL_IB_RETRY_CNT=13
NCCL_SOCKET_IFNAME=eth1

# GB10 측
LD_PRELOAD=/usr/lib/aarch64-linux-gnu/libnccl.so.2
NCCL_IB_HCA=rocep1s0f0 NCCL_IB_GID_INDEX=0 NCCL_IB_TIMEOUT=22 NCCL_IB_RETRY_CNT=13
NCCL_SOCKET_IFNAME=enp1s0f0np0
```

## SSH
- `ssh proxmox` → 192.168.1.4 (root, id_ed25519_ai_workbench)
- LXC 102 (100.x): `ssh -i ~/.ssh/id_ed25519_ai_workbench lcy2080@192.168.100.2`
- LXC 102 (1.x): `ssh -i ~/.ssh/id_3090_vm lcy2080@192.168.1.10` (키 등록 필요할 수 있음)
- pct exec: `ssh proxmox "pct exec 102 -- <command>"`
- VM 105: 삭제됨 (2026-03-16, GPU passthrough 충돌로 호스트 crash)

## opt_kernel 3090 빌드
```bash
# rsync (GB10 → 3090)
rsync -avz --exclude='.venv' --exclude='build' --exclude='*.egg-info' --exclude='__pycache__' --exclude='.git' \
  -e "ssh -i ~/.ssh/id_ed25519_ai_workbench" /home/lcy2080/ai/opt_kernel/ lcy2080@192.168.100.2:~/opt_kernel/
# 빌드
ssh -i ~/.ssh/id_ed25519_ai_workbench lcy2080@192.168.100.2 "cd ~/opt_kernel && .venv/bin/pip install -e ."
```

## 확정된 제한사항
- GPUDirect RDMA: GB10은 통합메모리라 GDR 불필요(RDMA=GPU접근), RTX 3090은 GeForce 드라이버 차단
- 3090 측 병목: GPU↔CPU 메모리 복사 (GDR 불가)
- RTX 3090 VM passthrough: GA102 reset bug
- ConnectX-3 ↔ ConnectX-7: RoCE v1만 동작 (포트 1)

## GDR 우회 시도 이력 (모두 실패 — 최종 확정)
- nvidia-peermem: 로드 성공, nvidia_p2p_get_pages() GeForce 차단
- 커스텀 커널 (peer_mem.ko, livepatch): 빌드/로드 성공, 동일 차단
- GDRCopy (2026-03-17): gdrdrv 빌드/로드 성공, gdr_pin_buffer → EINVAL(22), nvidia_p2p_get_pages() 동일 차단
- CUDA 속성 CU_DEVICE_ATTRIBUTE_GPU_DIRECT_RDMA_SUPPORTED = 0
- **결론: GeForce 드라이버는 nvidia_p2p_get_pages()를 펌웨어 레벨에서 차단, 모든 GDR 경로 불가**
- 리포: git.dan-lab.dev/lcy2080/MLNX_OFED_CUSTOM

## 향후
- ConnectX-5 도착 (~2주) → RoCE v2 안정성 개선
- DGX Spark 2대 직결: 200Gbps 가능 (공식 클러스터링 가이드 존재)
- Nemotron 3 Super BF16 + MXFP4_MOE 보유
