# 🖥️ VMware vSphere Nested 가상화 실습 랩 (Team1)

> **개인 학습 포트폴리오** | VMware Workstation 기반 중첩 가상화(Nested Virtualization) 환경 구축 및 vSphere 관리 실습

---

## 📌 프로젝트 개요

6대의 노트북을 활용하여 총 **12대의 ESXi 호스트**를 구축하고, **하나의 vCenter**로 통합 관리하는 대규모 가상화 랩(Lab) 환경을 구성하였습니다.

| 항목 | 내용 |
|------|------|
| **핵심 기술** | Nested Virtualization, VLAN/Subnet 분리, vMotion, Shared Storage (TrueNAS) |
| **하이퍼바이저** | VMware ESXi 7.0 |
| **관리 플랫폼** | VMware vCenter Server 7.0 |
| **네트워크 장비** | Aruba 2930F (L3 스위치), 일반 기가비트 스위치 |
| **스토리지** | TrueNAS (NFS / iSCSI) |
| **전체 호스트 수** | 12대 (노트북 6대 × ESXi 2대) |

---

## 🗂️ 목차

1. [팀 기본 세팅 (IP 할당 / vCenter 정보)](#1-팀-기본-세팅)
2. [아키텍처 구조도](#2-아키텍처-구조도)
3. [네트워크 구성 (Aruba Switch Topology)](#3-네트워크-구성)
4. [ESXi VM 생성 및 사양 설정](#4-esxi-vm-생성-및-사양-설정)
5. [ESXi 설치 및 IP 할당 (DCUI)](#5-esxi-설치-및-ip-할당-dcui)
6. [DNS 및 호스트명 설정](#6-dns-및-호스트명-설정)
7. [vCenter에 호스트 등록 및 DC 구성](#7-vcenter에-호스트-등록-및-dc-구성)
8. [ESXi 사용자 계정 생성 및 권한 부여](#8-esxi-사용자-계정-생성-및-권한-부여)
9. [잠금 모드 (Lockdown Mode)](#9-잠금-모드-lockdown-mode)
10. [Shared Datastore 종류 (NFS vs VMFS/iSCSI)](#10-shared-datastore-종류)

---

## 1. 팀 기본 세팅

### 팀원별 네트워크 IP 할당

> **네트워크 대역:** `10.10.0.0/16`

| 팀원 | ESXi 1 | ESXi 2 | Server |
|------|--------|--------|--------|
| 강민영 | 10.10.10.20 | 10.10.10.21 | 10.10.10.242 |
| 김민채 | 10.10.10.60 | 10.10.10.65 | 10.10.10.x |
| 김종연 | 10.10.10.30 | 10.10.10.31 | 10.10.10.x |
| 백주연 | 10.10.10.40 | 10.10.10.41 | 10.10.10.x |
| 이준호 | 10.10.10.10 | 10.10.10.15 | 10.10.10.241 |
| 이채유 | 10.10.10.50 | 10.10.10.55 | 10.10.10.245 |

### vCenter 접속 정보

| 항목 | 값 |
|------|----|
| 서버 주소 | `10.10.10.200` |
| 관리자 ID | `administrator@team1.vsphere` |
| 비밀번호 | `VMware1!` |

---

## 2. 아키텍처 구조도

```
┌─────────────────────────────────────────────┐
│           Physical Layer (Laptop)            │
│  CPU: Intel/AMD  RAM: 32GB+  Storage: SSD   │
│  Host OS: Windows / Host Hypervisor: WS     │
└────────────────────┬────────────────────────┘
                     │
┌────────────────────▼────────────────────────┐
│         VMware Workstation (Type 2)          │
│   Virtual Network 생성 / ESXi VM 실행       │
└──────────────┬──────────────┬───────────────┘
               │              │
   ┌───────────▼──┐    ┌──────▼────────┐
   │  ESXi Host A │    │  ESXi Host B  │
   │  RAM: 12 GB  │    │  RAM: 12 GB   │
   │  Disk1: 5GB  │    │  Disk1: 5GB   │
   │  Disk2: 30GB │    │  Disk2: 30GB  │
   │  NIC ×4      │    │  NIC ×4       │
   └──────────────┘    └───────────────┘

  노트북 1대당 위 구조 × 6대 = ESXi 12대 총합
```

### NIC별 네트워크 역할

| NIC | 네트워크 대역 | 용도 |
|-----|-------------|------|
| NIC 1 | `10.10.10.x` (Management) | ESXi 관리 및 호스트 접속 |
| NIC 2 | `10.10.20.x` (Storage) | TrueNAS iSCSI/NFS 스토리지 통신 |
| NIC 3 | `10.10.30.x` (VM Network) | 가상 머신 서비스 트래픽 |
| NIC 4 | `10.10.40.x` (vMotion) | 호스트 간 VM 실시간 이동 |

---

## 3. 네트워크 구성

### Aruba Switch Topology (물리적 연결)

```
[ 외부 네트워크 / 인터넷 ]
          │
          │ (Uplink)
[ Aruba 2930F Switch (L3 Core / Gateway: 10.10.10.1) ]
     /              \
[ ESXi 서버 ]   [ 일반 메인 스위치 (L2 Access) ]
(10.10.10.200)    / / │ │ \ \
                L1  L2  L3  L4  L5  L6
               (노트북 6대 연결)
```

### 연결 가이드

| 단계 | 내용 |
|------|------|
| 1단계 (Uplink) | Aruba 스위치 ↔ 일반 스위치 업링크 연결 (외부망 게이트웨이 역할) |
| 2단계 (서버) | 서버 NIC 1, 2, 3번 → 일반 스위치 포트 연결 |
| 2단계 (노트북) | 6대 노트북 → 일반 스위치 각 포트 연결 |
| 3단계 (vMotion) | NIC 4번 → 호스트 직접 연결 또는 고립된 별도 스위치 사용 |

### 이 구성의 장점

- **서버 성능 최적화**: 서버가 Aruba(코어)에 직접 연결되어 접근 단계 최소화
- **대역폭 확장**: 서버 트래픽이 일반 스위치를 우회해 병목 현상 감소
- **고립된 실습 환경**: Aruba에서 외부망만 차단하면 12대 호스트 간 완전 격리 실습 가능

---

## 4. ESXi VM 생성 및 사양 설정

### VMware Workstation 네트워크 사전 준비

1. `Edit > Virtual Network Editor` 실행
2. **VMnet0 (Bridged)**: 실제 NIC와 연결 (NIC 1, 2, 3 공용)
3. **VMnet1 (Host-only)**: 외부 차단 내부망 생성 (NIC 4 vMotion용)
   > ⚠️ `Use local DHCP service` 옵션 **반드시 해제** (IP는 수동 Static 할당)

### ESXi VM 사양

| 항목 | 설정값 | 비고 |
|------|--------|------|
| Guest OS | VMware ESXi 7 | Hardware Compatibility: ESXi 7.0 |
| CPU | 2 vCPU (소켓 2, 코어 2) | **Virtualize Intel VT-x/EPT 체크 필수** |
| RAM | 12 GB | vCenter 연결 및 실습 적정 사양 |
| Disk 1 | 5 GB (SCSI) | ESXi OS 설치용 |
| Disk 2 | 30 GB (SCSI) | Local Datastore용 |
| NIC 1, 2, 3 | Bridged (VMnet0) | 관리/스토리지/VM 트래픽 |
| NIC 4 | Host-only (VMnet1) | vMotion 전용 |

> 💡 **중첩 가상화 핵심**: VM Settings → Processors → `Virtualize Intel VT-x/EPT` 또는 `Virtualize AMD-V/RVI` 체크

---

## 5. ESXi 설치 및 IP 할당 (DCUI)

### 설치 순서

```
1. VMware Workstation에서 ESXi ISO 마운트 후 VM 부팅
2. ESXi 인스톨러 로드 완료 후 Enter
3. EULA 동의 → F11
4. 설치 디스크 선택 (5GB Disk 1) → Enter
5. 키보드 레이아웃: US Default
6. root 비밀번호 설정: VMware1!
7. 설치 확인 → F11
8. 설치 완료 후 재부팅 (ISO 언마운트)
```

### DCUI에서 IP 설정 (F2 로그인)

```
F2 → 비밀번호 입력 (VMware1!)
  → Configure Management Network
    → IPv4 Configuration
      → (3) Set static IPv4 address
        → IP:       10.10.10.xx
        → Subnet:   255.255.255.0
        → Gateway:  0.0.0.0 (NIC 1 게이트웨이는 상황에 따라 설정)
      → OK → Y (변경 사항 적용 및 재시작)
```

> ⚠️ **NIC 1(Management)에만** 게이트웨이 설정, NIC 2/3/4는 게이트웨이 없음

---

## 6. DNS 및 호스트명 설정

### DCUI에서 DNS 설정

```
F2 로그인
  → Configure Management Network
    → DNS Configuration
      → (o) Use the following DNS server addresses and hostname
        → Primary DNS Server: 10.10.10.2
        → Hostname:           JY-ESXi-01  (팀원별 명명 규칙 적용)
      → Enter (OK)
    → Custom DNS Suffixes
      → Suffixes: team1.com
      → Enter (OK)
  → Y (변경 사항 저장 및 재시작)
```

### 연결 확인

```bash
# Windows CMD에서 ping 테스트
ping esxi07.team1.com   # FQDN으로 확인
ping 10.10.10.40        # IP로도 확인
```

### 인프라 서비스 서버 구성 (중앙 서버)

| 서비스 | IP | 역할 |
|--------|----|------|
| vCenter Server | `10.10.10.200` | 12대 ESXi 통합 관리 및 클러스터링 |
| DNS 서버 (Windows) | `10.10.10.2` | FQDN 해석, vSphere 연동 |
| NAT 서버 | 중앙 서버 | 외부 인터넷 게이트웨이 |
| NTP 서버 | `10.10.10.2` | 전체 인프라 시간 동기화 |

---

## 7. vCenter에 호스트 등록 및 DC 구성

### 데이터센터(DC) 생성

```
vCenter 접속 (https://10.10.10.200)
  → vcsa.team1.com 우클릭
    → 새 데이터 센터
      → 이름: BJY-DC (팀원별 명명)
      → 위치: vcsa.team1.com
      → 확인
```

### 호스트 추가 (7단계)

| 단계 | 내용 |
|------|------|
| 1. 이름 및 위치 | ESXi IP (예: `10.10.10.40`) 입력, DC 선택 |
| 2. 연결 설정 | ID: `jy`, PW: `VMware1!` 입력 |
| 3. 호스트 요약 | 호스트 정보 확인 (VMware ESXi 7.0.3) |
| 4. 라이선스 할당 | 평가판 라이선스 선택 |
| 5. 잠금 모드 | **없음** 선택 (초기에는 비활성화 권장) |
| 6. VM 위치 | 해당 DC 선택 |
| 7. 완료 | 최종 설정 확인 후 완료 클릭 |

---

## 8. ESXi 사용자 계정 생성 및 권한 부여

> 단순히 계정을 만드는 것과 관리 권한을 부여하는 것은 **별개의 단계**입니다.

### 1단계: 사용자 계정 생성 (Create User)

```
https://esxi07.team1.com 접속 → root 로그인
  → 관리 (Manage)
    → 보안 및 사용자 (Security & users)
      → 사용자 (Users)
        → 사용자 추가 (Add user)
          → 사용자 이름: jy
          → 설명: juyeon
          → 암호: VMware1!
          → 추가
```

### 2단계: 권한 할당 (Assign Permissions)

```
왼쪽 최상단 → 호스트 (Host)
  → 작업 (Actions)
    → 권한 (Permissions)
      → 권한 추가 (Add permission)
        → 사용자 선택: jy
        → 역할 (Role): 관리자 (Administrator)
        → 확인
```

> ✅ 새로 만든 계정으로 ESXi Host Client 로그인 가능

---

## 9. 잠금 모드 (Lockdown Mode)

> **핵심 개념**: "이 호스트는 vCenter 말만 듣겠다!"  
> vCenter 이외의 경로를 통한 ESXi 직접 접근을 차단하여 보안을 강화합니다.

### 모드별 접근 제한 비교

| 접근 경로 | 기본 모드 (Disabled) | 일반 모드 (Normal) | 엄격 모드 (Strict) |
|-----------|---------------------|-------------------|--------------------|
| vCenter 관리 | ✅ 허용 | ✅ 허용 | ✅ 허용 |
| DCUI (물리 콘솔) | ✅ 허용 | ✅ 허용 | ❌ 차단 |
| SSH 접속 | ✅ 허용 | ❌ 차단 | ❌ 차단 |
| Host Client (웹 UI) | ✅ 허용 | ❌ 차단 | ❌ 차단 |

### 잠금 모드 설정 방법 (vCenter 기준)

```
vCenter → ESXi 호스트 선택
  → 구성 (Configure) 탭
    → 시스템 (System)
      → 보안 프로필 (Security Profile)
        → 잠금 모드 (Lockdown Mode) → 편집 (Edit)
          → 원하는 모드 선택 (Normal / Strict)
          → 확인 (즉시 적용)
```

### 예외 사용자 설정

잠금 모드 설정 시 **예외 사용자(Exception Users)** 를 등록하면 잠금 모드에서도 해당 계정으로 직접 접속 유지 가능합니다.

```
잠금 모드 설정 창 → 예외 사용자 탭
  → 사용자 추가 → 계정명 입력 (예: chaeyu)
```

> ⚠️ **Strict 모드 주의**: vCenter와 연결이 끊기면 누구도 해당 호스트에 접근 불가

---

## 10. Shared Datastore 종류

### NFS vs VMFS (iSCSI) 비교

| 항목 | NFS | VMFS (iSCSI) |
|------|-----|--------------|
| **스토리지 방식** | 파일 스토리지 | 블록 스토리지 |
| **파일 시스템 관리** | NAS가 관리 | ESXi가 VMFS 생성 |
| **ESXi 사용 방식** | 파일 직접 마운트 사용 | 스토리지가 LUN 제공 |
| **주요 용도** | ISO, Template 저장 | 여러 ESXi가 공유하는 VM 실행용 |
| **공유 실행** | 읽기 공유에 적합 | ✅ 여러 ESXi가 Shared Datastore로 VM 실행 가능 |

---

## 📁 프로젝트 주요 체크리스트

- [x] VMware Workstation 네트워크(VMnet) 구성 (VMnet0 Bridged, VMnet1 Host-only)
- [x] ESXi VM 생성 — CPU VT-x/EPT 중첩 가상화 옵션 활성화
- [x] ESXi 7.0 설치 (DCUI에서 Static IP, Subnet 설정)
- [x] DNS 서버 설정 및 호스트명(FQDN) 등록
- [x] vCenter에 전체 ESXi 호스트(12대) 등록
- [x] 데이터센터(DC) 생성 및 팀원별 구성
- [x] ESXi 사용자 계정 생성 + 관리자 권한 부여
- [x] 잠금 모드(Normal/Strict) 설정 및 테스트
- [x] Shared Datastore (NFS/VMFS iSCSI) 개념 정리
- [x] Aruba Switch 기반 물리 네트워크 토폴로지 구성

---

## 🔑 공통 자격증명 (실습 환경 전용)

| 구분 | 계정 | 비밀번호 |
|------|------|---------|
| ESXi root | `root` | `VMware1!` |
| vCenter 관리자 | `administrator@team1.vsphere` | `VMware1!` |
| 팀원 계정 예시 | `jy` | `VMware1!` |

---

## 📚 학습 키워드

`VMware` `ESXi` `vCenter` `vSphere` `Nested Virtualization` `중첩 가상화`  
`DCUI` `Lockdown Mode` `잠금 모드` `NFS` `iSCSI` `VMFS` `TrueNAS`  
`vMotion` `Shared Datastore` `DNS` `NTP` `Aruba Switch` `VLAN`  
`VMware Workstation` `Bridged Network` `Host-only Network`

---

## 🗒️ 참고 사항

- 본 실습은 **평가판 라이선스** 기준이며, 60일 후 만료됩니다.
- 모든 IP는 **수동(Static)** 으로 설정하며, DHCP는 비활성화합니다.
- vCenter 연동을 위해 각 ESXi의 **호스트명(FQDN)이 DNS에 등록**되어야 합니다.
- NIC 1(Management)에만 게이트웨이를 설정하고, NIC 2/3/4는 **게이트웨이 없음**으로 설정합니다.
