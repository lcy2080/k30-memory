---
name: migrate-proxmox-to-ubuntu-zfs-docker-k3s
description: Proxmox VE에서 Ubuntu Server + ZFS + Docker + K3s로 마이그레이션 계획
type: project
---

# 마이그레이션 개요

- **현재 환경**: Proxmox VE 9.1.6 (커널 6.17.13-2-pve)
 on AMD Threadripper 2990WX, 256GB RAM, RTX 3090
24GB VRAM
- **대상 환경**: Ubuntu Server 24.04 LTS + ZFS + Docker + K3s
- **하드웨어**: 동일 (물리 서버 그대로 사용)
- **마이그레이션 타입**: In-place (ZFS 풀 유지)

---

## 하드웨어 사양

### CPU
- **모델**: AMD Ryzen Threadripper 2990WX
32-Core Processor
- **코어/스레드**: 32C / 64T
- **NUMA 노드**: 4개 (0-7,32-39 / 16-23,48-55 | 8-15,40-47 | 24-31,56-63)
- **캐시**: L1d:1MiB, L1i:2MiB, L2:16MiB, L3:64MiB

- **RAM**: 256GB DDR4
- **스왕**: 없음

- **GPU**: NVIDIA RTX 3090 24GB VRAM (GA102)
- **NIC**: 다수 포트 + ConnectX-3 (40GbE, RoCE v1, RDMA)

- **스토리지**: ZFS 기반

| 풀 | 크기 | 타입 | 디스크 구성 | 사용량 |
|-----|------|------|------------------|
| rpool | 928GB | SSD | 1TB CT1000MX500 | 25.3GB | 2% |
| fast | 1.81TB | SSD | 2×2TB CT2000MX500 mirror | 80.3GB | 4% |
| tank | 21.8TB | HDD | 3×8TB HDD raidz1 + 2×500GB NVMe (log/cache) | 631GB | 3% |

---

## 현재 LXC 컨테이너



| ID | 이름 | vCPU | RAM | 디스크 | 마운트 | 서비스 |
|----|------|------|-----|-----|----------|---------|
| 100 | nas | 4 | 4GB | tank 4TB | - | Nextcloud AIO (Docker) |
| 101 | k3s-runner | 16 | 32GB | tank 1TB | - | K3s + GitLab Runner |
| 102 | ai-server | 32 | 64GB | fast 120GB | /models (tank/models) | AI 개발 환경 |
| 103 | git | 8 | 32GB | tank 1TB | - | GitLab CE (Omnibus) |
| 104 | nas-dev | 8 | 16GB | fast 50GB | - | PostgreSQL |

---

## 서비스별 상세 분석

---

## GPU 아키텍처 변경 (2026-03-17 결정)

### 기존: LXC GPU Passthrough
- LXC 102에 RTX 3090 전체 passthrough
- 단일 컨테이너만 GPU 사용 가능
- 관리 복잡도 높음

### 변경: Host GPU + Docker GPU 공유
- **Host**: NVIDIA 드라이버 직접 설치
- **Docker**: NVIDIA Container Toolkit으로 GPU 공유
- **장점**:
  - 여러 컨테이너가 GPU 동시 사용 가능 (MPS 또는 시분할)
  - 관리 단순화
  - K3s와 연동 용이 (nvidia-device-plugin)

### 필수 패키지
```bash
# NVIDIA 드라이버
sudo apt install nvidia-driver-570

# NVIDIA Container Toolkit
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt update && sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

### Docker GPU 사용 예시
```bash
docker run --gpus all nvidia/cuda:12.0-base-ubuntu22.04 nvidia-smi
```

---

## 테스트 결과 (VM 200 - 2026-03-17)

### 테스트 환경
- **VM ID**: 200
- **IP**: 192.168.1.24
- **리소스**: 32 vCPU / 64GB RAM
- **디스크**: scsi0 50GB (시스템) + scsi1/2 50GB×2 (ZFS)

### 완료된 항목
| 항목 | 상태 | 비고 |
|------|------|------|
| Ubuntu 24.04 LTS (UEFI) | ✅ | cloud-init 설치 |
| ZFS mirror 풀 (`datapool`) | ✅ | 48GB 사용 가능 |
| Docker 28.2.2 | ✅ | apt 설치 |
| K3s v1.34.5 | ✅ | control-plane Ready |
| PostgreSQL 마이그레이션 | ✅ | LXC 104 → Docker (mynas, mynas_test 복원) |

### 스냅샷
1. `base-config`: ZFS + Docker + K3s 기본 구성
2. `postgres-migrated`: PostgreSQL 마이그레이션 완료

### 마이그레이션 검증 명령
```bash
# ZFS 풀 확인
zpool status datapool
zfs list

# Docker 확인
docker ps
docker exec postgres-test psql -U postgres -c "\l"

# K3s 확인
kubectl get nodes
kubectl get pods -A
```

### SSH 접속
```bash
ssh -i ~/.ssh/id_ed25519_ai_workbench ubuntu@192.168.1.24
```

---

## 다음 단계

1. **ZFS 실제 풀 import**: 마이그레이션 당일 진행 (tank, fast, rpool)
2. **GPU 설정**: Host 드라이버 + NVIDIA Container Toolkit
3. **서비스 이관**:
   - GitLab (LXC 103) → Docker/K3s
   - Nextcloud (LXC 100) → Docker
   - AI 서비스 (LXC 102) → Docker + GPU 공유
4. **K3s 클러스터**: 필요시 worker 노드 추가

