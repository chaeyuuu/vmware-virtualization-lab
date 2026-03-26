# 🖥️ VMware vSphere 기반 사설 클라우드 인프라 구축

> 단일 장애점을 제거한 **고가용성 데이터센터**를 직접 설계·구축한 팀 프로젝트  
> ESXi 클러스터 · vCenter · 공유 스토리지 · HA/DRS · 고가용성 웹서비스까지 End-to-End 구현

![VMware](https://img.shields.io/badge/VMware_vSphere-607078?style=flat-square&logo=vmware&logoColor=white)
![Windows Server](https://img.shields.io/badge/Windows_Server-0078D4?style=flat-square&logo=windows&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=flat-square&logo=linux&logoColor=black)
![iSCSI](https://img.shields.io/badge/Protocol-iSCSI%20%7C%20NFS-grey?style=flat-square)

---

## 📐 Architecture

> *(아키텍처 구조도 이미지 삽입)*

---

## 👥 Team Members

| <img src="https://github.com/minykang.png" width="160px"> | <img src="https://github.com/minchaeki.png" width="160px"> | <img src="https://github.com/jongyeon0214.png" width="160px"> | <img src="https://github.com/juyeonbaeck.png" width="160px"> | <img src="https://github.com/Junhoss.png" width="160px"> | <img src="https://github.com/chaeyuuu.png" width="160px"> |
| :---: | :---: | :---: | :---: | :---: | :---: |
| [강민영](https://github.com/minykang) | [김민채](https://github.com/minchaeki) | [김종연](https://github.com/jongyeon0214) | [백주연](https://github.com/juyeonbaeck) | [이준호](https://github.com/Junhoss) | [이채유](https://github.com/chaeyuuu) |
---

## 📘 Index

- [01 — 인프라 설계 및 환경 세팅](#-day-01--인프라-설계-및-환경-세팅)
- [02 — 가상화 환경 구성 (ESXi + vCenter)](#-day-02--가상화-환경-구성-esxi--vcenter)
- [03 — 공유 스토리지 구성 (iSCSI / NFS)](#-day-03--공유-스토리지-구성-iscsi--nfs)
- [04 — 클러스터 고가용성 (HA / DRS)](#-day-04--클러스터-고가용성-ha--drs)
- [05 — 서비스 배포 및 운영](#-day-05--서비스-배포-및-운영)

---

## 🗓️ 01 — 인프라 설계 및 환경 세팅

> 전체 아키텍처 설계와 팀별 IP 대역 할당, 네트워크 토폴로지 확정

### 1. 목표

물리 서버 위에 ESXi 환경을 구성하고, 이후 모든 실습의 기반이 될 **네트워크 및 스토리지 설계**를 완성하는 것을 목표로 했습니다.

### 2. 주요 구성

- 팀별 IP 대역 및 vCenter 사전 환경 세팅
- Aruba Switch 기반 L2/L3 토폴로지 설계
- Management / Storage / vMotion 트래픽 분리 설계

### 3. 아키텍처 설계 결정

트래픽 간섭을 최소화하기 위해 VMkernel 어댑터를 역할별로 분리하고, 각 호스트가 전용 포트 그룹을 통해서만 해당 트래픽을 처리하도록 설계했습니다.

---

## 🖥️ 02 — 가상화 환경 구성 (ESXi + vCenter)

> ESXi 호스트 설치부터 vCenter 클러스터 등록까지

### 1. ESXi 설치 및 네트워크 설정 (DCUI)

USB 부팅으로 ESXi를 설치하고, DCUI 인터페이스를 통해 Management Network IP를 수동 할당했습니다.

### 2. DNS 등록 및 vCenter 배포

- DNS 서버 구축 후 vCenter FQDN 레코드 사전 등록
- vCenter Server Appliance (vCSA) 배포 및 SSO 도메인 구성

### 3. vCenter 클러스터 구성

- 호스트 3대를 vCenter에 등록하고 Datacenter / Cluster 생성
- ESXi 사용자 계정 생성, 역할 기반 권한 부여
- Lockdown Mode 설정 (Normal / Strict 비교 검토)

> **Lockdown Mode 핵심:** vCenter를 통한 중앙 관리만 허용하고, ESXi 직접 접속(Host Client, SSH)을 차단하여 보안 우회를 방지합니다.

| 모드 | DCUI | Host Client | vCenter |
|------|------|-------------|---------|
| 비활성 | ✅ | ✅ | ✅ |
| Normal | 예외 사용자만 | ❌ | ✅ |
| Strict | ❌ | ❌ | ✅ |

### ⚠️ Troubleshooting

> 관련 트러블슈팅은 [Troubles 폴더](./Troubles)를 참고하세요.

---

## 💾 03. 공유 스토리지 구성 (iSCSI 기반 VMFS)


vSphere 클러스터의 핵심 기능인 **vMotion**과 **HA(High Availability)**를 구현하기 위해, Windows Server를 활용한 iSCSI 공유 스토리지 환경을 구축했습니다.

---

### 1. 개요 및 의사결정 (Storage Strategy)

단순한 파일 공유 방식인 NFS와 블록 스토리지 방식인 iSCSI 중, 가상화 환경에 최적화된 **iSCSI(VMFS)**를 선택하여 구축했습니다.

- **선택 이유:** VMFS(Virtual Machine File System)는 다중 호스트가 동일한 LUN에 동시 접근할 때 데이터 무결성을 보장하는 **Distributed Locking** 메커니즘을 지원하며, 고성능 I/O 처리에 유리합니다.
- **할당 용량:** Windows Server 내 300GB VHDX 생성 및 할당.

---

### 2. Windows Server: iSCSI Target 설정

Windows Server를 '하드디스크를 빌려주는 주인'으로 만드는 과정입니다.

### 📍 Step 1: iSCSI 대상 서버 역할 설치

1. **서버 관리자** 실행 → [관리] → [역할 및 기능 추가].
2. `파일 및 저장소 서비스` > `파일 및 iSCSI 서비스` > **[iSCSI 대상 서버]** 체크 후 설치.

### 📍 Step 2: iSCSI 가상 디스크 및 대상(Target) 지정

1. **가상 디스크 생성:** 300GB 용량의 `.vhdx` 파일을 생성합니다.
2. **접근 권한(IQN) 등록:** 각 ESXi 호스트의 고유 식별자인 **IQN**을 등록하여 지정된 호스트만 스토리지를 점유할 수 있도록 보안을 설정합니다.
    
    > **IQN이란?** `iqn.1998-01.com.vmware:esxi-01` 처럼 IP가 바뀌어도 장치를 식별할 수 있는 고유 주소입니다.
    > 

---

### 3. ESXi: iSCSI Initiator 연결 및 VMFS 구성

각 ESXi 호스트에서 네트워크를 통해 300GB 디스크를 '내 것'처럼 인식시키는 과정입니다.

### 📍 Step 1: 어댑터 활성화 및 네트워크 바인딩

- [스토리지] > [어댑터] > **[소프트웨어 iSCSI 추가]**를 클릭합니다.
- **Port Binding:** 스토리지 전용 VMkernel 포트를 어댑터에 바인딩하여 관리 트래픽과의 간섭을 차단합니다.

### 📍 Step 2: Dynamic Discovery & Rescan

- Windows Server의 IP(**10.10.10.2**)를 대상으로 등록하고 '다시 검사(Rescan)'를 수행합니다.
- 정상적으로 연결되면 300GB 용량의 **MSFT iSCSI Disk**가 장치 목록에 나타납니다.

### 📍 Step 3: VMFS 6 데이터스토어 생성

인식된 디스크를 vSphere 표준 파일 시스템으로 포맷합니다.

---

## ⚙️ 04 — 클러스터 고가용성 (HA / DRS)

> 장애 자동 복구와 리소스 로드밸런싱으로 무중단 운영 환경 구성

### 1. HA (High Availability)

ESXi 호스트 장애 감지 시 해당 호스트의 VM을 클러스터 내 다른 호스트에서 자동 재시작합니다.

- Admission Control 정책으로 장애 대응 여유 리소스 예약
- VM 재시작 우선순위 및 모니터링 설정

### 2. DRS (Distributed Resource Scheduler)

클러스터 내 ESXi 호스트 간 부하를 지속 모니터링하고, 리소스 불균형 발생 시 VM을 자동 마이그레이션합니다.

- 자동화 수준: Manual / Partially Automated / Fully Automated
- Affinity / Anti-Affinity 규칙으로 VM 배치 정책 제어

### 3. Resource Pool

- 팀/서비스 단위로 CPU·메모리 자원을 논리적으로 격리
- Shares / Reservation / Limit 3단계 정책 적용

### 4. vCenter 백업 (VAMI)

- `https://{vcenter-ip}:5480` 접근 → 백업 스케줄 설정
- FTP 서버를 백업 위치로 지정하여 정기 자동 백업

---

## 🚀 05 — 서비스 배포 및 운영

> 표준화된 템플릿 기반 VM 배포와 고가용성 웹서비스 구축

### 1. Linux 마스터 템플릿 제작

반복 배포를 위한 읽기 전용 마스터 이미지를 제작했습니다.

```bash
# OS 식별 정보 초기화 (Ubuntu 기준)
apt clean && apt autoremove -y
cloud-init clean
rm -f /etc/ssh/ssh_host_*
truncate -s 0 /etc/machine-id
cat /dev/null > ~/.bash_history && history -c && poweroff
```

- vCenter 사용자 지정 규격으로 배포 시 IP / 호스트명 자동 주입
- VMware Tools 설치 필수

### 2. NFS 기반 고가용성 웹서비스 구축

- NFS 공유 스토리지에 웹 콘텐츠를 올려 모든 웹 서버 VM이 동일한 데이터를 바라보도록 구성
- ESXi HA와 결합하여 노드 장애 시 서비스 자동 복구

### 3. NTP 서버 동기화

- Windows Server를 NTP 서버로 구성
- 전체 ESXi 호스트 시간 동기화 적용

---

## 🛠️ 기술 스택

| 분류 | 기술 |
|------|------|
| 하이퍼바이저 | VMware ESXi 8.x |
| 가상화 관리 | VMware vCenter Server |
| 공유 스토리지 | Windows Server iSCSI Target |
| 스토리지 프로토콜 | iSCSI (VMFS), NFS |
| 네트워크 | Aruba Switch, VMkernel, vSwitch |
| OS | Windows Server 2022, Ubuntu Server |
| 기타 | VAMI 백업, NTP, Lockdown Mode |

---

## 📁 전체 챕터 목록

<details>
<summary>펼치기 (25개)</summary>

| # | 제목 |
|---|------|
| 01 | 팀 기본 세팅 (IP 할당 / vCenter 정보) |
| 02 | 아키텍처 구조도 |
| 03 | 네트워크 구성 (Aruba Switch Topology) |
| 04 | ESXi VM 생성 및 사양 설정 |
| 05 | ESXi 설치 및 IP 할당 (DCUI) |
| 06 | DNS 및 호스트명 설정 |
| 07 | vCenter에 호스트 등록 및 DC 구성 |
| 08 | ESXi 사용자 계정 생성 및 권한 부여 |
| 09 | 잠금 모드 (Lockdown Mode) |
| 10 | Shared Datastore 종류 (NFS vs VMFS/iSCSI) |
| 11 | 가상화 인프라 구성 요소 (Who's who?) |
| 12 | ESXi 호스트별 네트워크 설정 (VMkernel / vSwitch) |
| 13 | vSphere 공유 스토리지 유형 상세 정리 |
| 14 | 데이터스토어 자동 전파 원리 (3가지 이유) |
| 15 | vCenter SSH 활성화 (VAMI) |
| 16 | Windows Server NTP 서버 설정 |
| 17 | Windows Server iSCSI Target 설정 (2단계) |
| 18 | ESXi iSCSI 연결 및 VMFS 데이터스토어 생성 |
| 19 | ESXi 가상 네트워크 계층 구조 상세 |
| 20 | vCenter 백업 설정 (FTP 서버 활용) |
| 21 | 클러스터 설정 — DRS |
| 22 | 클러스터 설정 — HA |
| 23 | Resource Pool 설정 및 활용 |
| 24 | Linux 마스터 템플릿 제작 및 배포 |
| 25 | NFS 기반 고가용성 공유 웹 서비스 구축 |

</details>

---

*본 프로젝트는 실습 환경에서 진행된 인프라 구축 기록입니다.*
