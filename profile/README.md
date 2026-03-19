# 🌐 FWS (Four Guys Web Service)

## 1. 📖 Overview
FWS (Four Guys Web Service)는 VMware vSphere 기반 가상화 인프라 위에서
가상머신을 온디맨드로 생성하고, 웹 브라우저를 통해 즉시 접속할 수 있도록 제공하는
가상머신 프로비저닝 플랫폼이다.

사용자는 별도의 복잡한 설정 없이 웹 인터페이스에서
OS, CPU, Memory, Storage를 선택하여 VM을 생성할 수 있으며,
Apache Guacamole을 통해 브라우저에서 바로 SSH 접속이 가능하다.

사용이 끝난 VM은 반납 시 자동으로 삭제되어
클러스터 자원을 효율적으로 재사용할 수 있다.

본 시스템은 vCenter, ESXi, vSAN, vMotion, DRS, HA 기반의 가상화 인프라 위에 구축되었으며,  
프라이빗 클라우드 형태의 VM 운영 환경 제공을 목표로 한다.

---

## 2. 🧩 Key Features
### ⚡ On-Demand VM Provisioning
필요한 순간에 가상머신을 즉시 생성하고 사용자 요구에 맞게 자원을 할당할 수 있다.

---

### 🌐 Browser-Based Remote Access
Apache Guacamole을 통해 웹 브라우저만으로 VM에 바로 SSH 접속이 가능하다.

---

### 🔄 Full VM Lifecycle Management
VM 생성 → 접속 → 반납까지의 전체 Lifecycle을 하나의 웹 서비스에서 통합 관리한다.

---

### 🧠 Cluster-Based Resource Optimization
VMware Cluster 환경에서 DRS, vMotion을 활용하여 리소스를 자동으로 분산하고 효율적으로 관리한다.

---

### 🛡️ High Availability Infrastructure
HA 기반으로 구성되어 Host 장애 발생 시 VM을 자동으로 복구하여 서비스 가용성을 유지한다.

---

### 📦 Distributed Shared Storage with vSAN
vSAN을 기반으로 여러 ESXi Host의 로컬 스토리지를 통합하여  
클러스터 전체에서 사용할 수 있는 분산형 공유 스토리지를 구성한다.

---

### 📝 Centralized Log Storage with NFS
로그 데이터를 중앙에서 관리하고 영속적으로 저장한다.

---

### 🌍 Network Segmentation Design
트래픽을 역할별로 분리하여 성능과 보안을 동시에 확보한다.

---

## 3. 🏗️ System Architecture

본 시스템은 다음과 같은 계층 구조로 구성된다.

- **Web Layer**: 사용자 인터페이스 제공
- **API Layer**: VM 생성 및 관리 로직 처리
- **Virtualization Layer**: vCenter / ESXi 기반 VM 운영
- **Infrastructure Layer**: vSAN, 네트워크, 클러스터 구성

![System Architecture](./images/architecture.png)

---
## 4. 🌐 Network Architecture

본 시스템은 관리, 외부 접속, 사용자 트래픽, 스토리지, 마이그레이션, vSAN, 로그 트래픽을 분리하기 위해 역할별로 vSwitch와 Port Group을 구성하였다.

### Network Segments

| Port Group | CIDR | Purpose |
|------|------|------|
| Internet | 192.168.0.0/24 | 외부 접속 및 서비스 노출 |
| NTP | 172.18.0.0/20 | 시간 동기화 및 기본 인프라 통신 |
| Shared-Storage | 172.18.16.0/20 | 스토리지 전용 네트워크 |
| VMotion | 172.18.32.0/20 | VM 라이브 마이그레이션 트래픽 |
| VM-User-Traffic | 172.18.48.0/20 | 사용자 VM 서비스 트래픽 |
| vSAN-Network | 172.18.64.0/20 | vSAN 클러스터 내부 통신 |
| LogDB | 172.18.80.0/20 | 로그 수집 및 관리 트래픽 |

---

### Virtual Switch Layout

| vSwitch | Connected Port Group | Role |
|------|------|------|
| vSwitch0 | Management Network, NTP | ESXi 관리 및 기본 인프라 서비스 |
| vSwitch-internet | Internet | 외부 접속 및 인터넷 연결 |
| vSwitch1 | Shared-Storage | 스토리지 전용 통신 |
| vSwitch2 | VM-User-Traffic | 사용자 VM 및 서비스 트래픽 |
| vSwitch3 | VMotion | ESXi Host 간 VM 마이그레이션 |
| vSwitch4 | vSAN-Network | vSAN 클러스터 내부 통신 |
| vSwitch5 | LogDB | 로그 저장 및 수집용 네트워크 |

---

### Why Network Segmentation?

네트워크를 역할별로 분리한 이유는 다음과 같다.

- 관리 트래픽과 사용자 트래픽 간 간섭 방지
- 스토리지 및 클러스터 통신 성능 확보
- 장애 발생 시 영향 범위 최소화
- 보안성과 운영 가시성 향상
- VMware 기능(vMotion, vSAN, HA, DRS)의 안정적 동작 보장

![Network Architecture](./images/network-architecture.png)

---

## 5. 🖥️ Virtual Machine Design

본 시스템은 역할별로 VM을 분리하여 구성하였으며,  
각 VM은 특정 기능을 담당하고 해당 기능에 맞는 네트워크에 연결되어 있다.  

이를 통해 서비스 계층, 관리 계층, 스토리지 계층을 분리하여  
확장성과 안정성을 확보하였다.

---

### Service Server (Web + API)
- 사용자 요청 처리 (VM 생성 / 반납)
- vCenter API 호출을 통한 VM 프로비저닝
- 로그 데이터를 LogDB로 전송

### Guacamole Server
- 브라우저 기반 SSH 접속 제공
- 사용자와 VM 간 연결 게이트웨이

### vCenter Server
- ESXi Host 및 VM 관리
- 클러스터 기능 제어 (DRS, HA, vMotion)

### NAT Server
- 내부 VM의 외부 인터넷 통신 지원

### DNS / NTP Server
- DNS: 이름 기반 통신 지원
- NTP: 시간 동기화

### NFS Server
- LogDB 데이터 영속성 저장소

### LogDB Server
- 로그 수집 API (FastAPI)
- PostgreSQL 기반 로그 저장 (Docker)
- NFS를 통한 데이터 영속성 확보

---

## 6. ⚙️ Infrastructure Design & Technology Choices

본 시스템은 **사용자가 VM을 요청하는 순간부터 반납까지의 전체 라이프사이클을 안정적으로 운영**하는 것을 목표로 설계되었다.  
단순한 VM 생성 도구가 아니라, 프라이빗 클라우드 수준의 자원 관리와 장애 대응이 가능한 환경을 구성하기 위해 다음 기술들을 선택하였다.

### Cluster + DRS — 자원 분산 자동화

사용자가 VM을 생성할 때마다 특정 ESXi Host에 자원이 집중되는 문제를 방지하기 위해 DRS를 활성화하였다.

DRS는 클러스터 내 CPU와 Memory 사용량을 지속적으로 분석하여, 부하가 집중된 Host의 VM을 vMotion을 통해 다른 Host로 자동 이동시킨다.  
이를 통해 관리자가 직접 VM 배치를 조정하지 않아도 클러스터 전체의 자원 균형을 유지할 수 있다.

---

### HA (High Availability) — Host 장애 대응

ESXi Host 장애 발생 시 해당 Host 위의 VM을 다른 Host에서 자동으로 재시작하여 서비스 가용성을 유지하도록 구성하였다.

또한 HA Failover가 정상적으로 수행되지 않을 경우 vCenter 알람을 통해 관리자에게 즉시 통보되도록 설정하였다.  
장애를 자동으로 감지하는 것뿐 아니라, 복구 실패 상황까지 빠르게 인지할 수 있어야 운영 안정성을 확보할 수 있다고 판단했기 때문이다.

---

### vSAN — 공유 스토리지 없이 HA / vMotion 구성

HA와 vMotion이 정상적으로 동작하려면 모든 ESXi Host가 동일한 VM 데이터에 접근할 수 있어야 한다.  
별도의 외부 스토리지 장비 없이 이를 구현하기 위해 vSAN을 선택하였다.

vSAN은 각 ESXi Host의 로컬 디스크를 클러스터 단위로 묶어 분산형 공유 스토리지를 구성한다.  
이를 통해 추가 스토리지 장비 없이도 HA, vMotion, DRS가 동작할 수 있는 환경을 구축하였다.

---

### vMotion — 무중단 VM 이동

vMotion은 DRS의 자동 재배치 과정과 Host 유지보수 시 실행 중인 VM을 중단 없이 다른 Host로 이동시키는 역할을 한다.

vSAN 기반 공유 스토리지를 사용하기 때문에 VM 디스크 자체를 이동하지 않고도 실행 상태를 다른 Host로 이전할 수 있어, 서비스 중단 없이 운영과 유지보수가 가능하다.

---

### NFS — 로그 데이터 영속성 확보

LogDB 서버는 PostgreSQL을 Docker 컨테이너로 운영하고 있으며, 컨테이너 재시작 시 데이터 유실 가능성이 존재한다.  
이를 방지하기 위해 PostgreSQL의 데이터 디렉토리(`pgdata`)를 NFS 마운트로 연결하였다.

이 구조를 통해 컨테이너가 재시작되거나 LogDB VM에 장애가 발생하더라도 로그 데이터는 NFS에 영구적으로 저장되도록 구성하였다.

---

### Network Segmentation — 트래픽 격리

vSAN, vMotion, 사용자 트래픽이 동일한 네트워크를 공유하면 대역폭 경합이 발생하고 전체 성능에 영향을 줄 수 있다.  
이를 방지하기 위해 각 트래픽을 전용 vSwitch와 Port Group으로 분리하였다.

특히 vSAN과 vMotion은 대량의 데이터를 전송하는 작업이므로, 사용자 서비스 트래픽과 분리하여 운영하는 것이 성능 확보와 안정성 측면에서 중요하다.

---

### Summary

| Component | Role |
|------|------|
| vSAN | Distributed Shared Storage |
| NFS | Log Storage |
| vMotion | Live VM Migration |
| Cluster | Resource Pool |
| DRS | Load Balancing |
| HA | Failure Recovery |
| NAT | External Network Access |
| DNS | Name Resolution |
| NTP | Time Synchronization |

---

## 7. 🔄 VM Provisioning Flow

### 1. VM 생성
- OS / CPU / Memory / Storage 선택
- VM 생성 요청 → vCenter API 호출
- VM 생성 완료

### 2. VM 접속
- VM 이름 입력
- Guacamole을 통해 SSH 접속
- 인증 후 접속 완료

### 3. VM 반납
- VM 이름 입력
- 검증 후 삭제 진행
- 자원 회수 완료

![Flowchart](./images/flowchart.png)

---
## 8. 🔐 Remote Access Architecture

본 시스템은 사용자가 별도의 SSH 클라이언트 없이  
웹 브라우저만으로 가상머신에 접속할 수 있도록  
**Apache Guacamole** 기반의 원격 접속 구조를 사용한다.

Apache Guacamole은 브라우저와 내부 VM 사이에서 SSH 세션을 중계하는 역할을 수행한다.  
이를 통해 사용자는 별도의 SSH 클라이언트 없이 웹 브라우저만으로 VM에 접속할 수 있다.

### Architecture Overview

원격 접속 구조는 다음과 같은 흐름으로 동작한다.

```text
User Browser
     │
     ▼
Apache Guacamole Web
     │
     ▼
guacd
     │
     ▼
SSH
     │
     ▼
Client VM
```
---

## 9. 🛠️ Tech Stack

### Virtualization
- VMware vSphere: vSphere Client 버전 7.0.3.01900
- ESXi: VMware ESXi 7.0
- vCenter  7.0.3

### Backend
- FastAPI 

### Remote Access
- Apache Guacamole 1.6.0

### Database
- MySQL 8.0
- PostgreSQL 16

### Storage
- vSAN (Distributed Storage) 
- NFS (Log Storage) 

### Infrastructure Features
- vMotion 
- DRS 
- HA 

---

## 10. 📂 Project Structure

```bash
.
├── frontend/          # Web Service (HTML, CSS, JS)
├── backend/           # FastAPI 서버
├── scripts/           # VM 생성 / 관리 스크립트
├── images/            # 아키텍처 및 다이어그램
└── README.md
```

---

## 11. 🚀 Deployment

본 시스템은 VMware 기반 가상화 환경 위에 구성되며 다음과 같은 순서로 구축된다.

1. ESXi Host 설치 및 네트워크 구성
2. vCenter Server 배포 및 클러스터 구성
3. vSAN 구성 (스토리지 통합)
4. Network Segmentation 설정 (vSwitch / Port Group)
5. 인프라 VM 구성 (DNS, NTP, NAT, NFS)
6. 서비스 VM 배포 (Web, FastAPI, Guacamole)
7. VM 템플릿 생성 및 자동화 구성

---

## 12. 📊 Future Work

- 사용자 인증 및 권한 관리 기능 추가
- VM 사용량 기반 과금 시스템 설계
- Kubernetes 기반 컨테이너 환경 확장
- 모니터링 시스템 (Prometheus, Grafana) 도입
- 로그 분석 및 자동화 시스템 구축
- Auto Scaling 및 정책 기반 자원 할당
---

## 13. 👥 Team
- 팀 소개
- 역할 분담

---

## 14. 📌 References
- 사용 기술 문서
- 참고 자료


