# 🖥️ VMware vSphere Nested 가상화 실습 랩 (Team1)

> **개인 학습 포트폴리오** | VMware Workstation 기반 중첩 가상화(Nested Virtualization) 환경 구축 및 vSphere 관리 실습

---

## 📌 프로젝트 개요

6대의 노트북을 활용하여 총 **12대의 ESXi 호스트**를 구축하고, **하나의 vCenter**로 통합 관리하는 대규모 가상화 랩(Lab) 환경을 구성하였습니다.

| 항목 | 내용 |
|------|------|
| **핵심 기술** | Nested Virtualization, VLAN/Subnet 분리, vMotion, Shared Storage (TrueNAS / Windows iSCSI) |
| **하이퍼바이저** | VMware ESXi 7.0 |
| **관리 플랫폼** | VMware vCenter Server 7.0 |
| **네트워크 장비** | Aruba 2930F (L3 스위치), 일반 기가비트 스위치 |
| **스토리지** | TrueNAS (NFS / iSCSI), Windows Server iSCSI Target |
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
11. [가상화 인프라 구성 요소 (Who's who?)](#11-가상화-인프라-구성-요소)
12. [스토리지 VM 권장 설정 (TrueNAS / RAID-5)](#12-스토리지-vm-권장-설정)
13. [ESXi 호스트별 네트워크 설정 (VMkernel / vSwitch)](#13-esxi-호스트별-네트워크-설정)
14. [vSphere 공유 스토리지 유형 상세 정리](#14-vsphere-공유-스토리지-유형-상세-정리)
15. [ZFS / ZVOL 개념 정리 (TrueNAS)](#15-zfs--zvol-개념-정리)
16. [데이터스토어 자동 전파 원리 (3가지 이유)](#16-데이터스토어-자동-전파-원리)
17. [vCenter SSH 활성화 (VAMI)](#17-vcenter-ssh-활성화-vami)
18. [오픈 소스 스토리지 솔루션 비교 (TOP 3)](#18-오픈-소스-스토리지-솔루션-비교-top-3)
19. [TrueNAS 설치 및 초기 설정 (Step-by-Step)](#19-truenas-설치-및-초기-설정)
20. [TrueNAS ZFS 계층 구조 및 Pool 생성](#20-truenas-zfs-계층-구조-및-pool-생성)
21. [TrueNAS VDEV 옵션 상세 가이드](#21-truenas-vdev-옵션-상세-가이드)
22. [Windows Server NTP 서버 설정](#22-windows-server-ntp-서버-설정)
23. [Windows Server iSCSI Target 설정 (2단계)](#23-windows-server-iscsi-target-설정)
24. [ESXi iSCSI 연결 및 VMFS 데이터스토어 생성](#24-esxi-iscsi-연결-및-vmfs-데이터스토어-생성)
25. [ESXi 가상 네트워크 계층 구조 상세](#25-esxi-가상-네트워크-계층-구조-상세)
26. [vCenter 백업 설정 (FTP 서버 활용)](#26-vcenter-백업-설정)
27. [클러스터 설정 — DRS (Distributed Resource Scheduler)](#27-drs-distributed-resource-scheduler)
28. [클러스터 설정 — HA (High Availability)](#28-ha-high-availability)
29. [Resource Pool 설정 및 활용](#29-resource-pool-설정-및-활용)
30. [Linux 마스터 템플릿 제작 및 배포](#30-linux-마스터-템플릿-제작-및-배포)

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
> vCenter를 통하지 않고 호스트(ESXi)에 직접 들어오는 문을 어디까지 열어둘 것인가를 결정합니다.

### 모드별 접근 제한 전체 비교

| 접근 경로 | Disabled (기본) | Normal (일반) | Strict (엄격) |
|-----------|:--------------:|:------------:|:------------:|
| vCenter 관리 | ✅ 가능 | ✅ 가능 (필수) | ✅ 가능 (필수) |
| ESXi 웹/SSH 접속 | ✅ 허용 | ❌ 차단 | ❌ 차단 |
| DCUI (물리 콘솔) | ✅ 허용 | ✅ 허용 (예외 사용자만) | ❌ 차단 |
| 보안 수준 | 낮음 (기본 상태) | 높음 (권장) | 매우 높음 (위험 동반) |
| 특징 | 누구나 관리자 계정으로 접속 가능 | vCenter가 죽어도 본체에서 복구 가능 | vCenter가 죽으면 호스트 접속 불가 |

### 각 모드 설명

- **Disabled**: 실습 환경에서 쓰는 가장 편한 상태. Host Client, SSH 모두 `root` 로그인 가능
- **Normal**: 기업 환경 표준. 원격 웹/SSH 접속은 막아 해킹 위험 감소, 서버 본체 DCUI 비상구는 유지
- **Strict**: 국방·금융권 수준. DCUI마저 완전 차단. **vCenter 없으면 호스트 관리 방법 전무**

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

### 예외 사용자 (Exception User)

예외 사용자는 잠금 모드가 활성화된 상태에서도 **호스트에 직접 로그인할 수 있는 권한**을 유지하는 특별 계정입니다.

| 잠금 모드 | 일반 사용자 (Admin 포함) | 예외 사용자 |
|-----------|------------------------|------------|
| Disabled | 웹·SSH·DCUI 모두 자유 접속 | 의미 없음 (모두가 예외와 동일) |
| Normal | vCenter 통해서만 관리 가능 | **DCUI(서버 본체) 접속 가능** |
| Strict | 모든 직접 접속 차단 (DCUI 포함) | **DCUI(서버 본체) 접속 가능 (유일한 비상구)** |

```
잠금 모드 설정 창 → 예외 사용자 탭
  → 사용자 추가 → 계정명 입력 (예: chaeyu, jy)
```

### 실습 검증 결과 (Strict + 예외 사용자 jy 설정 시)

| 테스트 | 결과 |
|--------|------|
| `root` → Host Client 접속 | ❌ 거부 ("이 작업을 수행할 수 있는 사용 권한이 거부되었습니다") |
| 예외 사용자 `jy` → Host Client 접속 | ✅ 정상 접속 |
| `root` → SSH 접속 | ❌ `Connection closed by 10.10.10.41 port 22` |
| 예외 사용자 `jy` → SSH 접속 | ✅ `[jy@esxi08:~]` 정상 접속 |

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

## 11. 가상화 인프라 구성 요소

> **비유**: ESXi는 '아파트 건물', VM들은 '각각의 세대'.
> 한 건물 안에 편의점(DNS), 창고(Storage), 관리사무소(vCenter)가 모두 공존합니다.

### 각 구성 요소 역할

| 구성 요소 | 별칭 | 역할 |
|-----------|------|------|
| **ESXi** | The Foundation / Hypervisor | 물리 하드웨어(CPU·RAM·Disk)를 쪼개 VM에게 나눠주는 **집주인** |
| **DNS Server** | The Directory / Address Book | `esxi01.team1.com` → `10.10.10.11` 로 연결하는 **주소록** |
| **TrueNAS (datastore-server)** | The Storage / Warehouse | 모든 서버가 공동으로 쓰는 거대한 **공용 창고**. RAID-5로 데이터 안전 보관 |
| **vCenter** | The Manager | 여러 ESXi 호스트를 한곳에서 관리하는 **관리소장** |

### 인프라 계층 구조

```
[ 3층: 가상 머신 계층 (Virtual Machines) ]
  🏠 VM A (10.10.10.2):   Windows Server → DNS/NTP 역할
  🏢 VM B (10.10.10.200): vCenter Server → 관리소장 역할

[ 2층: 하이퍼바이저 계층 (Hypervisor) ]
  🛠 VMware ESXi: 물리 자원을 쪼개 VM들에게 나눠주는 OS

[ 1층: 물리 서버 계층 (Physical Hardware) ]
  🖥 서버 본체: 실제 CPU, RAM, Disk가 들어있는 장비
```

---

## 12. 스토리지 VM 권장 설정

### TrueNAS VM 기본 사양 (VMware Workstation 기준)

| 항목 | 권장 설정 | 비고 |
|------|-----------|------|
| CPU | 2 vCPUs | 스토리지 처리에 CPU 연산 필요 |
| RAM | 8 GB 이상 | TrueNAS 권장 (ZFS 캐시용) / OMV는 2~4GB도 가능 |
| NIC | VMXNET3 | VMware 환경에서 가장 빠른 어댑터 타입 |

### 디스크 구성 (RAID-5 기준)

> **RAID-5 원리**: N개의 디스크 중 1개 분량을 패리티(복구용 데이터) 저장에 사용
> **용량 계산**: `(N-1) × Disk Size` → 예) `(4-1) × 100GB = 300GB`

| 구분 | 개수 | 개별 용량 | RAID-5 실사용 용량 | 용도 |
|------|------|-----------|-------------------|------|
| OS용 디스크 | 1개 | 16~32 GB | — | TrueNAS 시스템 설치 |
| Virtual Disk (SSD 타입) | 4개 | 100 GB | **300 GB** | 시스템 OS 및 고성능 데이터 |
| Virtual Disk (HDD 타입) | 4개 | 100 GB | **300 GB** | SMB, NFS 공유 폴더용 |

> 💡 VMware Workstation에서 디스크 추가 시 `Virtual disk type`을 `SATA` 또는 `NVMe`로 설정하면 시스템이 SSD처럼 인식합니다.

---

## 13. ESXi 호스트별 네트워크 설정

### IP 할당 대역 정리 (12대 호스트 기준)

| 대역 (Subnet) | 용도 | 호스트별 IP 범위 | 주요 서버 |
|--------------|------|----------------|----------|
| `10.10.10.x` | 관리 (Management) | .10 ~ .52 | vCenter, NTP, DNS |
| `10.10.20.x` | 스토리지 (iSCSI) | .10 ~ .52 | Windows iSCSI 서버 (.2) |
| `10.10.30.x` | 이사 (vMotion) | .10 ~ .52 | 호스트 간 통신 전용 |

### vmnic vs VMkernel (VMK) 개념

| 구분 | vmnic (물리적 입구) | VMkernel (논리적 서비스 통로) |
|------|-------------------|------------------------------|
| 정체 | 서버 뒷면의 **실제 랜카드** | ESXi가 통신하기 위해 만든 **가상 인터페이스** |
| IP 주소 | ❌ 없음 (그냥 통로) | ✅ 있음 (`10.10.20.11` 같은 주소가 여기에 붙음) |
| 비유 | 집으로 오는 **물리 인터넷 선** | 거실의 **컴퓨터 / IPTV 셋톱박스** |
| 관계 | 데이터가 나가는 **길(Physical)** | 데이터를 보내는 **주체(Logical)** |

```
vmnic (물리 NIC)
  → vSwitch (가상 스위치, 다리 역할)
    → VMkernel (서비스용 번호판 = IP 주소)
```

### VMkernel 어댑터 설정 절차 (각 호스트 반복)

**1단계: vmnic 확인**

```
각 ESXi VM의 vmnic0 ~ vmnic3 정상 인식 확인
  vmnic0: 관리용 (Management, 이미 사용 중)
  vmnic1: 스토리지용 (10.10.20.x)
  vmnic2: vMotion용 (10.10.30.x)
  vmnic3: 가상 머신 트래픽용
```

**2단계: vSwitch 생성**

```
[네트워킹 추가] → [표준 스위치의 가상 머신 포트 그룹]
  → [새 표준 스위치 만들기]
    → 해당 대역 vmnic 할당
      예) 스토리지 스위치 → vmnic1 할당
    → 이름 지정: vSwitch-Storage / vSwitch-vMotion
```

**3단계: VMkernel 어댑터(vmk) 생성**

```
[네트워킹 추가] → [VMkernel 네트워크 어댑터]
  → 위에서 만든 vSwitch 선택
  → 포트 속성 설정:
      네트워크 레이블: Storage-Network 또는 vMotion-Network
      사용 가능 서비스:
        - 20번 대역(Storage): 체크박스 모두 해제 (iSCSI는 나중에 바인딩)
        - 30번 대역(vMotion): [vMotion] 반드시 체크!
  → IPv4 설정: 10.10.20.xx / 255.255.255.0 입력
```

**4단계: iSCSI 어댑터와 VMkernel 바인딩 (스토리지 전용)**

```
[스토리지 어댑터] → [iSCSI Software Adapter]
  → 하단 탭 [네트워크 포트 바인딩(Network Port Binding)]
    → [추가(+)] → 스토리지용 VMkernel (vmk1) 선택 등록
```

---

## 14. vSphere 공유 스토리지 유형 상세 정리

### 스토리지 유형 3가지

| 유형 | 프로토콜 레벨 | 특징 | vSphere 주요 용도 |
|------|-------------|------|------------------|
| **SMB** | 파일 수준 | Windows 환경 파일 공유 프로토콜 | 특정 앱·리포지토리 용도 (VM 데이터스토어로는 비권장) |
| **NFS** | 파일 수준 | NAS 장비와 통신. 설정 간단 | ISO, Template 저장, 실습·중소규모 인프라 |
| **VMFS on iSCSI** | 블록 수준 | TCP/IP로 하드디스크 통째 빌림 | VM 실행용 Shared Datastore, 대규모 엔터프라이즈 |

### NFS vs iSCSI 심층 비교

| 구분 | NFS (파일 수준) | iSCSI (블록 수준) |
|------|----------------|-----------------|
| **파일 시스템** | NFS 서버가 관리 | ESXi가 직접 VMFS로 포맷 |
| **설정 난이도** | ✅ 쉬움 (마운트 방식) | ⚠️ 복잡 (IQN, LUN, 포트 바인딩 필요) |
| **확장성** | 용량 확장이 매우 유연 | LUN 크기 확장이 상대적으로 번거로움 |
| **CPU 부하** | 상대적으로 낮음 | 데이터 처리량 많을 때 높음 |
| **Thin Provisioning** | 스토리지 차원에서 기본 지원 | VMFS 차원에서 지원 |
| **연결 전파** | 각 호스트마다 개별 마운트 필요 | 한 곳에서 포맷 시 다른 호스트 자동 인식 |
| **Windows 서버 입장** | "내 폴더 안 파일들이 보이네" (관리 가능) | "큰 파일 하나가 있긴 한데 내용물은 안 보이네?" |
| **ESXi 서버 입장** | "남의 집 폴더를 연결해서 쓰자" | "내 몸에 붙은 내 하드디스크다!" |

### iSCSI 저장소의 실체

```
[Windows Server (10.10.10.2)]
  └── .vhdx 파일 (가상 디스크 파일) 생성
        └── iSCSI 프로토콜로 ESXi에게 LUN 전달
              ↓
[ESXi 호스트]
  └── "새 하드디스크가 꽂혔다!" 라고 인식
        └── VMFS로 즉시 포맷 → Datastore 사용 시작
```

> **LUN (Logical Unit Number)**: iSCSI를 통해 네트워크로 전달되는 가상 디스크 단위

---

## 15. ZFS / ZVOL 개념 정리

### ZFS (Zettabyte File System)

> **한 줄 정의**: 파일 시스템(FileSystem) + 볼륨 매니저(Volume Manager)가 합쳐진 스마트 관리자

- **기존 방식**: 디스크를 묶는 기술(RAID)과 파일을 저장하는 기술(FileSystem)이 별도로 동작
- **ZFS 방식**: 혼자서 디스크를 묶어 **Pool(창고)** 을 만들고, 데이터 무결성을 스스로 검사·수정 (**Self-healing**)

> 🏭 비유: 단순한 상자가 아닌, 물건이 상했는지 감시하고 자동으로 분류해주는 **스마트 물류 센터**

### ZVOL (ZFS Volume)

> **한 줄 정의**: ZFS Pool 안에 만든 **가상의 하드디스크**

- ZFS 창고(Pool) 안에 공간을 떼어 "이건 그냥 빈 하드디스크처럼 써!"라고 내어주는 방식
- **iSCSI 블록 스토리지 서비스 시 필수**. ESXi는 이 ZVOL을 받아 VMFS로 포맷

> 🔒 비유: 큰 창고 안에 칸막이를 쳐서 만든 **독립된 개인 금고** (밖에서 보면 그냥 Raw Disk)

### Dataset vs ZVOL

| 구분 | **Dataset (데이터셋)** | **ZVOL (ZFS 볼륨)** |
|------|----------------------|---------------------|
| 형태 | 일반적인 **폴더** 형태 | 가상의 **디스크** 형태 |
| 주요 서비스 | **NFS, SMB** (파일 공유) | **iSCSI** (블록 공유) |
| 관리 주체 | TrueNAS가 파일 관리 | 연결해 쓰는 쪽(ESXi)이 관리 |
| 비유 | 서랍장 안의 **폴더** | 서랍장 안에 들어있는 **하드디스크** |

```
✅ NFS 실습  → Dataset 생성
✅ iSCSI 실습 → ZVOL 생성
```

---

## 16. 데이터스토어 자동 전파 원리

> **현상**: esxi-01에서 Datastore를 만들었는데, esxi-02 ~ esxi-12에도 자동으로 나타났다!
> → 이건 고장이 아니라 **공유 스토리지(Shared Storage)가 정상 동작하는 증거**입니다.

### 자동 전파가 일어나는 3가지 이유

**이유 1. Access Control — 입장 허가 목록 등록**

iSCSI Target에 모든 ESXi의 IP(또는 IQN)를 등록했기 때문입니다.

```
스토리지 서버 입장: "01번부터 12번까지 모두 이 하드디스크를 쓸 VIP들이구나!"
  → 모든 호스트가 같은 문(네트워크)을 통해 같은 방(LUN)을 바라보는 상태
```

**이유 2. VMFS는 클러스터 파일 시스템**

일반 NTFS·EXT4는 한 번에 한 컴퓨터만 연결 가능하지만, **VMFS는 여러 대가 동시에 마운트 가능**하도록 설계되었습니다.

```
01번 호스트가 "이 디스크를 VMFS로 포맷해!" 명령
  → 물리 디스크 헤더에 "여기는 VMFS 창고" 표식이 새겨짐
  → 02~12번 호스트들이 표식 발견 → 자동으로 Mount!
```

**이유 3. vCenter의 Rescan 자동화**

```
vCenter가 중간에서 신호 발송:
"01번이 새 창고 만들었으니까 나머지도 얼른 가서 확인해!"
  → 12번까지 일일이 [새로고침] 누르지 않아도 자동으로 목록에 등재
```

### NFS는 왜 호스트마다 수동으로 해줘야 하나?

| 구분 | iSCSI (VMFS) | NFS |
|------|-------------|-----|
| 핵심 개념 | 하드디스크를 통째로 빌려옴 | 서버의 특정 폴더를 빌려옴 |
| 포맷 주체 | ESXi가 직접 VMFS로 포맷 | 스토리지 서버가 이미 포맷 (NTFS 등) |
| 연결 전파 | 한 곳에서 포맷 → 다른 호스트 **자동 인식** | 각 호스트가 서버 폴더 주소를 **일일이 등록** 필요 |

> 💡 단, vCenter의 **'모든 호스트에 마운트'** 기능을 사용하면 NFS도 한 번에 전체 등록 가능

---

## 17. vCenter SSH 활성화 (VAMI)

> vCenter의 SSH는 보안상 **기본적으로 비활성화**되어 있습니다.
> VAMI(VMware Appliance Management Interface)를 통해 활성화합니다.

### 활성화 절차

```
접속 주소: https://10.10.10.200:5480
계정: root / (vCenter 설치 시 설정한 비밀번호)
  → 좌측 메뉴 [액세스 (Access)]
    → [편집 (Edit)] 클릭
      → 'SSH 로그인 활성화 (Enable SSH Login)' 토글 → ON
      → (선택) 'Bash 쉘 활성화'도 함께 켜두면 편리
      → [확인 (OK)] 저장
```

```bash
# SSH 접속 확인
ssh root@10.10.10.200

# Appliance Shell에서 Bash로 전환
shell
```

### ESXi 호스트에서 SSH 서비스 시작

```
vCenter → ESXi 호스트 선택
  → [구성] → [시스템] → [서비스]
    → SSH → [시작] 클릭
      (상태: 실행 중 확인)
```

---

## 18. 오픈 소스 스토리지 솔루션 비교 (TOP 3)

### 솔루션 개요

#### 1. TrueNAS (Core / Scale) — "가장 강력한 끝판왕"

가장 유명하고 널리 쓰이는 솔루션으로 기업에서도 실제 백업 서버나 파일 서버로 많이 사용합니다.

- **특징**: ZFS라는 아주 강력한 파일 시스템을 사용해서 데이터 보호 능력이 뛰어납니다.
- **vSphere 호환성**: iSCSI와 NFS 성능이 매우 안정적이라 VMware 실습용으로 가장 많이 추천됩니다.

#### 2. OpenMediaVault (OMV) — "가볍고 친절한 데비안 기반"

데비안 리눅스 기반의 경량 NAS 솔루션입니다.

- **특징**: 성능이 가볍고 웹 UI가 직관적입니다. 라즈베리 파이 같은 저사양 기기에서도 잘 돌아갑니다.
- **장점**: 필요한 기능만 플러그인 형태로 설치해서 쓸 수 있어 관리가 매우 깔끔합니다.

#### 3. StarWind SAN & NAS — "VMware 실습의 단짝"

오픈 소스는 아니지만, 무료 버전(Free Edition)이 강력해서 VMware 사용자들 사이에서 필수템으로 꼽힙니다.

- **특징**: 특히 iSCSI 성능에 특화되어 있습니다. 윈도우용 소프트웨어도 있어서 설치가 매우 쉽습니다.
- **장점**: 2대의 서버를 하나처럼 묶는 HA 기능을 실습하기에 최적화되어 있습니다.

### 솔루션 한눈에 비교

| 이름 | 기반 OS | 난이도 | 추천 용도 |
|------|---------|--------|----------|
| **TrueNAS** | FreeBSD/Debian | 중 | 고성능, 기업급 데이터 관리 |
| **OMV** | Debian Linux | 하 | 가벼운 실습, 리눅스 학습 병행 |
| **StarWind** | Windows/Linux | 하 | VMware 클러스터, 고가용성(HA) 실습 |

> 📥 TrueNAS 다운로드: https://www.truenas.com/download/

---

## 19. TrueNAS 설치 및 초기 설정

### 1단계: 가상 머신(VM) 생성 (VMware Workstation)

TrueNAS를 설치하기 전, 먼저 VM 그릇을 잘 만들어야 합니다.

| 항목 | 설정값 | 비고 |
|------|--------|------|
| OS Type | Other Linux 5.x kernel 64-bit (또는 Debian 11/12) | — |
| CPU | 2 vCPUs | — |
| RAM | **8GB 이상** | ZFS 캐시를 위해 8GB 이상을 강력 권장 |
| OS용 디스크 | 16GB ~ 32GB | TrueNAS 시스템이 설치될 공간 |
| 데이터용 (SSD 그룹) | 100GB × 4개 | RAID-Z1 구성용 |
| 데이터용 (HDD 그룹) | 100GB × 4개 | RAID-Z1 구성용 |
| Network | ESXi들과 통신 가능한 VMnet | 예: `10.10.10.x` 대역 |

### 2단계: 설치 프로세스

```
1. TrueNAS ISO 파일을 마운트하고 부팅
2. 메뉴에서 [1. Install/Upgrade] 선택
3. 드라이브 선택: 가장 작은 용량(16~32GB) 디스크 선택
   (나머지 100GB 8개는 나중에 '풀'을 만들 때 사용)
4. root 계정의 비밀번호 설정
5. 부팅 모드: 실습 환경이므로 BIOS 모드 선택 가능
6. 설치 완료 후 ISO 제거 → 재부팅
```

### 3단계: 웹 UI 접속 및 초기 네트워크 설정

재부팅 후 콘솔 화면 메뉴:

```
1) Configure network interfaces   → IP 설정
2) Configure network settings     → 게이트웨이/DNS 설정
3) Configure static routes
4) Set up local administrator
5) Reset configuration to defaults
6) Open TrueNAS CLI Shell
7) Open Linux Shell
8) Reboot
9) Shutdown
```

IP 설정 예시 (`10.10.10.110/24` 할당 후 웹 브라우저로 접속):

```
http://10.10.10.110  또는  https://10.10.10.110
→ 만들었던 아이디(truenas_admin), 비밀번호로 로그인
```

### TrueNAS 버전 선택 가이드

| 사용자 유형 | TrueNAS Enterprise | TrueNAS Community Edition |
|------------|-------------------|--------------------------|
| 개발자 | 이용 불가 | 26일 밤 (최신 빌드) |
| 얼리 어답터 | 이용 불가 | 25.10.2.1 |
| 일반적인 | 25.10.1 | 25.10.1 |
| 임무 필수 | 25.04.2.6 ★ | 기업 |

---

## 20. TrueNAS ZFS 계층 구조 및 Pool 생성

### ZFS 3단계 계층 구조

TrueNAS의 저장소는 다음과 같은 3단계 층으로 구성됩니다.

```
[ Disk (물리 디스크) ]
  └── 100GB짜리 하드디스크들 (물리 또는 가상 디스크)
        │
[ VDEV (가상 장치, Virtual Device) ]
  └── 디스크들을 RAID-Z1(RAID5) 등으로 묶은 '단위'
        │
[ Pool (저장소 풀) ]
  └── VDEV들을 하나 이상 합쳐서 만든 최종적인 거대한 저장 공간
```

> ⚠️ **주의**: 만약 VDEV 하나가 통째로 깨지면, 그 VDEV가 속한 **전체 Pool의 데이터가 날아갑니다**.
> 따라서 VDEV를 구성할 때 신중해야 합니다.

### Pool 생성 절차 (Storage > Create Pool)

**1단계: Pool 이름 설정**

```
Storage → Create Pool
  → Pool 이름 입력 (예: iSCSI_pool, NFS_pool)
  → non-unique serial 디스크 사용 시 → [Allow] 선택 (미선택 시 디스크 목록 미표시)
  → [Next]
```

**2단계: Data VDEV 구성**

```
Data VDEV 설정:
  - Layout: RAIDZ1  ← RAID5와 동일한 방식
  - Disk Size: 100 GiB (SSD)
  - Width: 4         ← VDEV 하나에 들어갈 디스크 수
  - Number of VDEVs: 1
  → [Save And Go To Review]
```

**Width와 Number of VDEVs 설정 가이드**

| 설정 | 설명 |
|------|------|
| **Width** | 하나의 VDEV(디스크 묶음)에 들어갈 디스크의 개수. RAID-5를 원하면 4 선택 |
| **Number of VDEVs** | 그런 묶음을 몇 개 만들어서 합칠 것인가. 4개를 한 덩어리로 끝낼 것이므로 1 선택 |

**용량 계산 (수학적 근거)**

```
설정: Width 4 / RAID-Z1
용량 계산: (Width - 1) × 디스크 용량 = 사용 가능한 용량
결과: (4 - 1) × 100GB = 300GB

⚠️ 만약 Width 2 / VDEVs 2로 설정하면, 2개씩 묶인 RAID-1(Mirror) 2개가
   합쳐져서 전혀 다른 구조가 되어버리니 주의!
```

**3단계: Log / Spare / Cache / Metadata / Dedup — 모두 Skip**

실습 환경에서는 모두 기본값(Skip)으로 두고 진행합니다. (상세 내용은 다음 섹션 참조)

**4단계: Review & Create**

```
Review 화면에서 최종 확인:
  - Pool Name: iSCSI_pool
  - Topology Summary: Data: 1 × RAIDZ1 | 4 × 100 GiB (SSD)
  - Estimated Usable Capacity: 300 GiB
  → [Create Pool] 클릭
  → "The contents of all added disks will be erased." 경고 → [Confirm] → [Continue]
```

**실전 팁 — SSD Pool과 HDD Pool 분리**

```
SSD 4개 + HDD 4개를 가지고 있을 때:
  1. SSD용 Pool을 만들 때: Width 4 / VDEVs 1
  2. HDD용 Pool을 만들 때: Width 4 / VDEVs 1

각각 따로따로 풀을 만들어야 속도 차이에 따른 병목 현상이 생기지 않습니다.
```

---

## 21. TrueNAS VDEV 옵션 상세 가이드

### Log (Optional) — ZIL (ZFS Intent Log)

> **역할**: 쓰기 속도를 높이기 위해 별도의 빠른 SSD를 캐시로 쓰는 설정

| 구분 | 설명 |
|------|------|
| ZIL이란? | 쓰기 명령을 고속 SSD에 먼저 기록해두고 나중에 실제 스토리지에 반영하는 방식 |
| 실습 환경 권장 | ⛔ **Skip** — 별도의 캐시용 디스크를 추가하지 않았으니 무시하고 넘어갑니다. |

### Spare (Optional) — Hot Spare

> **역할**: RAID에서 디스크 하나가 고장 나면 즉시 투입되어 자동으로 데이터 복구(Rebuild)를 시작하는 대기 디스크

| 구분 | 설명 |
|------|------|
| Hot Spare | 전원이 켜진 채로 대기하고 있다가 바로 작동 |
| 실습 환경 비권장 이유 | Spare 설정 시: 3개 데이터용 + 1개 Spare → 가용 용량이 200GB로 감소. `(3-1) × 100GB = 200GB` |
| 결론 | ⛔ **Skip** — 4개를 모두 Data VDEV에 사용해서 300GB 확보하는 것이 이득 |

### Cache (Optional) — L2ARC

> **역할**: 자주 읽는 데이터를 RAM보다 크고 일반 HDD보다 빠른 고성능 SSD에 미리 저장해두었다가 빠르게 읽어오는 기술

| 구분 | 설명 |
|------|------|
| L2ARC란? | Level 2 Adaptive Replacement Cache. RAM(ARC)에 없는 데이터를 SSD에서 빠르게 서빙 |
| 성능 가속 | 느린 HDD까지 가지 않고 캐시용 SSD에서 바로 응답 |
| 실습 환경 비권장 이유 | 캐시(L2ARC) 목록 관리에 실제 RAM을 일정량 소모. RAM이 부족하면 오히려 전체 속도가 더 느려질 수 있음 |
| 결론 | ⛔ **Skip** — 별도의 고속 SSD가 없다면 굳이 설정 불필요 |

### Metadata (Optional) — Fusion Pool

> **역할**: 메타데이터(파일의 위치 정보)와 아주 작은 파일들을 별도의 빠른 전용 SSD에 따로 저장

| 구분 | 설명 |
|------|------|
| Fusion Pool | 일반 느린 HDD(데이터)와 빠른 SSD(메타데이터)를 섞어서 만든 풀 |
| 성능 가속 | 파일 목록 조회 시 느린 HDD를 뒤질 필요 없이 전용 SSD에서 바로 응답 |
| 치명적 위험성 | 메타데이터 전용 디스크가 고장 나면 → **데이터 디스크가 멀쩡해도 전체 Pool 데이터를 전혀 읽을 수 없음** |
| 결론 | ⛔ **Skip** — 기초 단계에서는 건너뛰는 것이 안전 |

### Dedup (Optional) — 중복 제거

> **역할**: 똑같은 데이터 블록이 들어오면 실제로 저장하지 않고 기존 위치만 가리켜서 저장 공간을 절약

| 구분 | 설명 |
|------|------|
| 원리 | 중복 데이터 주소록(DDT, Deduplication Table)을 유지하며 중복 블록 참조 처리 |
| "RAM 도둑" | 주소록 확인에 엄청난 RAM 소모. 데이터 1TB당 약 1~5GB의 RAM 필요 |
| 성능 저하 | RAM 부족 시 디스크에서 주소록을 읽어야 해 시스템 속도가 극도로 느려짐 |
| 결론 | ⛔ **Skip** — 실습 규모(300GB)에서는 이득보다 손해가 훨씬 큼 |

---

## 22. Windows Server NTP 서버 설정

> ESXi 호스트 간의 시간 동기화는 클러스터 운영(vMotion, HA 등)에 필수적입니다.
> Windows Server를 내부 NTP 서버로 설정하여 모든 ESXi 호스트가 동기화되도록 합니다.

### 설정 절차

**1단계: 레지스트리 수정 — NTP 서버 활성화**

```
Win + R → regedit 실행
  → 경로: HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\TimeProviders\NtpServer
    → [Enabled] 값을 1로 변경
```

**2단계: 레지스트리 수정 — 공고 플래그 설정**

```
  → 경로: HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\Config
    → [AnnounceFlags] 값을 5로 변경
       (자체 시간 소스로 신뢰하도록 설정)
```

**3단계: Windows Time 서비스 재시작**

```powershell
net stop w32time
net start w32time
```

**4단계: 방화벽 설정**

```
제어판 → Windows 방화벽 → 고급 설정
  → 인바운드 규칙 → 새 규칙
    → UDP 포트 123 허용
```

**5단계: 설정 확인 (선택)**

```cmd
w32tm /query /configuration
```

출력 예시:

```
[Configuration]
EventLogFlags: 2 (Local)
AnnounceFlags: 5 (Local)
...
[TimeProviders]
NtpClient (Local): NTP Type, InputProvider: 1
NtpServer (Local): InputProvider: 0, Enabled: 1
```

---

## 23. Windows Server iSCSI Target 설정

> Windows Server를 'iSCSI 하드디스크를 빌려주는 주인(Target)'으로 설정합니다.

### 1단계: iSCSI 대상 서버 역할 추가

```
서버 관리자(Server Manager) 실행
  → 오른쪽 상단 [관리(Manage)] → [역할 및 기능 추가]
    → 마법사에서 '서버 역할' 단계까지 다음 클릭
    → [파일 및 저장소 서비스] 항목 펼치기 (>)
      → [파일 및 iSCSI 서비스] 항목 한 번 더 펼치기
        → [iSCSI 대상 서버] 체크 → 설치
```

### 2단계: iSCSI 가상 디스크 생성

```
[파일 및 저장소 서비스] → [iSCSI] → [iSCSI 가상 디스크 만들기] 클릭
  → 저장 경로 지정 (예: E:\ 드라이브 → 300GB 여유 공간)
  → 용량 설정 (예: 100GB)
    → 파일은 .vhdx 형식으로 저장됩니다.
```

### 3단계: iSCSI 대상(Target) 생성 및 IQN 등록

> **IQN (iSCSI Qualified Name)**이란?
> iSCSI 장치마다 부여되는 절대 변하지 않는 고유 이름입니다.

**IQN 형식**:

```
iqn.1998-01.com.vmware:esxi-01-2a3b4c5d

iqn       : iSCSI 이름을 뜻하는 고정 문구
1998-01   : 해당 표준이 정의된 연도와 월
com.vmware: 도메인 이름을 거꾸로 쓴 것 (식별용)
esxi-01...: 사용자가 지정하거나 시스템이 자동 생성한 고유 이름
```

**IQN 등록 절차**:

```
1. ESXi(Initiator)에서 확인:
   각 ESXi 호스트 설정 화면 → 스토리지 어댑터 → iSCSI Software Adapter
   → 자신의 IQN 주소 확인 및 복사

2. Windows(Target)에 등록:
   iSCSI 대상 설정 창 → "액세스할 수 있는 Initiator" 목록
   → ESXi의 IQN 붙여넣기 등록

3. 결과:
   Windows Server가 등록된 IQN을 가진 ESXi에만 디스크 공유
```

---

## 24. ESXi iSCSI 연결 및 VMFS 데이터스토어 생성

### 1단계: 소프트웨어 iSCSI 어댑터 추가

```
vSphere Client → ESXi 호스트 선택
  → [구성] → [스토리지] → [스토리지 어댑터]
    → [소프트웨어 어댑터 추가] → [소프트웨어 iSCSI 추가]
      → [확인]
```

### 2단계: 동적 검색(Dynamic Discovery) — 대상 서버 등록

```
iSCSI Software Adapter 선택
  → 하단 탭 [동적 검색]
    → [대상 추가] 클릭
      → iSCSI 서버 IP: 10.10.10.2 (Windows iSCSI 서버)
        포트: 3260 (기본값)
      → [확인]
```

> ✅ 등록 후 [새로 고침]을 누르면 Windows에서 만든 디스크가 'Unconsumed' 상태로 표시됩니다.

### 3단계: 스토리지 재검색 (Rescan)

```
[스토리지 어댑터] → [스토리지 다시 검색] 아이콘 클릭
→ [모든 스토리지 어댑터 재검색] 선택
```

### 4단계: VMFS 데이터스토어 생성 (5단계)

```
1. 메뉴 진입:
   [호스트 및 클러스터] → [ESXi 호스트] → [스토리지] → [새 데이터스토어]

2. 유형 선택:
   [VMFS] 선택 → [다음]

3. 이름 및 장치 선택:
   이름: iSCSI-Storage-01 (명명 규칙에 따라)
   장치: 목록에서 300GB짜리 MSFT iSCSI Disk 선택

4. 버전 선택:
   [VMFS 6] 선택 (최신 기능과 대용량을 지원하는 표준 버전)

5. 파티션 구성:
   [모든 사용 가능한 공간 사용] 선택 → [마침]
```

### 동적 검색 대상 완벽하게 제거하는 법

잘못 등록된 iSCSI 대상을 삭제할 때는 순서가 중요합니다.

```
1. [정적 검색 (Static Discovery)] 탭으로 이동
   → 해당 IP와 관련된 항목들을 먼저 모두 [제거(Remove)]
   (자식 항목을 먼저 삭제해야 부모 항목 삭제 가능)

2. [동적 검색 (Dynamic Discovery)] 탭으로 복귀
   → 해당 IP 항목 선택 → [제거] 클릭

3. [스토리지 재검색 (Rescan Storage)] 클릭
   → '모든 스토리지 어댑터 재검색' 아이콘 클릭
```

### 무한 로딩 탈출법 (응급 처치)

**방법 1: 호스트 관리 에이전트 재시작 (가장 효과적)**

```bash
# ESXi 호스트에 SSH로 접속 후:
/etc/init.d/hostd restart
/etc/init.d/vpxa restart
```

결과: vCenter와의 연결이 잠시 끊겼다 다시 붙으면서, 꽉 막혀있던 '제거 작업'이 강제로 종료됩니다.

**방법 2: iSCSI 어댑터 설정 초기화**

```
[스토리지 어댑터] → [iSCSI Software Adapter] 선택
  → [사용 안 함 (Disable)] 클릭
⚠️ 주의: 이미 연결된 다른 정상 데이터스토어가 있다면 서비스 중단 가능
```

**방법 3: 브라우저 새로고침**

```
F5 누르거나 로그아웃 후 재접속
  → '최근 작업' 탭이 깨끗해졌는지 확인
```

---

## 25. ESXi 가상 네트워크 계층 구조 상세

### 네트워크 데이터 흐름

데이터가 밖으로 나가는 흐름을 이해하는 것이 가장 중요합니다.

```
VMkernel (IP 부여) → vSwitch (가상 스위치) → vmnic (물리/가상 랜카드)
```

| 구성 요소 | 역할 | 특징 |
|----------|------|------|
| **VMkernel (vmk)** | ESXi의 서비스 통로 | IP 주소가 실제로 부여되는 곳. 관리/스토리지/vMotion 용도별로 생성 |
| **vSwitch** | 내부 가상 장치들과 외부 물리 선을 연결해 주는 가상 허브 | 여러 VMkernel과 Port Group을 연결 |
| **vmnic** | 서버 뒷면의 실제 랜선 구멍 (또는 가상화된 NIC) | 외부망으로 나가는 최종 출입구 |

### 3대 전용 도로 설계 (Subnet Planning)

트래픽 간섭을 방지하기 위해 용도별로 도로를 분리합니다.

| 대역 | 용도 | 비고 |
|------|------|------|
| `10.10.10.x` | 관리 (Management) | vCenter, SSH, Host Client 접속 |
| `10.10.20.x` | 스토리지 (Storage/iSCSI) | TrueNAS, Windows iSCSI 서버 통신 |
| `10.10.30.x` | vMotion | 호스트 간 VM 라이브 마이그레이션 |

### vSwitch 생성 절차 (Step-by-Step)

**Step 1: 새 vSwitch 생성 (전용 도로 개설)**

```
vCenter 접속 → 호스트 선택 → [구성] → [네트워킹] → [가상 스위치]
  → [네트워킹 추가] 클릭
    → [표준 스위치의 가상 머신 포트 그룹] 선택
    → [새 표준 스위치 만들기] 선택 → 미사용 중인 vmnic1 할당
    → 이름(레이블) 설정: vSwitch-Storage (뒤에 용도 suffix 추가 권장)
    → [완료]
```

### VLAN ID 개념 및 설정

> **VLAN ID란?**
> 서브네팅이 논리적 구분이라면, VLAN ID는 물리 선(vmnic)이 하나일 때 데이터에 이름표를 붙여서 트래픽을 확실히 분리하는 **태깅(Tagging)** 기술입니다.

| 상황 | 설정 방법 |
|------|----------|
| **물리 스위치 VLAN 설정이 없을 때** (가장 일반적) | VLAN ID를 `0` 혹은 `None`으로 설정. 이 상태를 **Access Mode (Untagged)** 라고 부름 |
| **물리 스위치에서 VLAN을 나눴을 때** | 스위치에서 설정한 VLAN 번호와 동일하게 입력. 예) 스위치에서 VLAN 20 → ESXi도 VLAN ID: 20 |

### Step 2: VMkernel 어댑터 추가 (번호판 달기)

```
[네트워킹 추가] → [VMkernel 네트워크 어댑터] 선택
  → 위에서 만든 vSwitch-Storage 선택
  → 포트 속성:
      네트워크 레이블: VMkernel Storage
      VLAN ID: 없음(0)
      서비스 체크박스: 모두 해제 (스토리지 전용이므로)
        ※ vMotion 용 VMkernel이라면 [vMotion] 체크
  → IPv4 설정:
      IP 주소: 10.10.20.41  (해당 호스트의 전용 IP)
      서브넷 마스크: 255.255.255.0
      기본 게이트웨이: 체크 해제 (스토리지/vMotion은 같은 대역 내 통신이므로 불필요)
```

**"재정의(Override)"는 언제 쓰나요?**

- **체크 안 할 때**: 호스트의 전체 기본 게이트웨이(주로 10번 대역)를 따라갑니다. 스토리지나 vMotion은 같은 대역 내에서만 통신하기 때문에 게이트웨이가 따로 없어도 됩니다.
- **체크할 때**: 스토리지 서버가 아주 먼 다른 네트워크(L3 너머)에 있어서, 꼭 이 어댑터만의 전용 출구가 필요할 때만 씁니다. 지금의 실습 환경에서는 **체크 해제가 정석**입니다.

### Step 3: iSCSI 타겟 연결 (네트워크 포트 바인딩)

```
[스토리지 어댑터] → [iSCSI Software Adapter] 선택
  → 하단 [네트워크 포트 바인딩] 탭
    → 위에서 만든 스토리지용 vmk 어댑터를 선택하여 묶어줌

  → [동적 검색] 탭 → [추가]
    → 스토리지 서버 IP: 10.10.20.2 (Windows iSCSI 또는 TrueNAS IP) 입력
```

> 💡 **"사용되지 않음" 표시의 의미**: VMkernel 생성 시 서비스 체크박스를 모두 비워두었기 때문에 표시되는 것입니다. iSCSI 통신은 이 '서비스' 체크박스를 통해 이루어지는 게 아니라, [네트워크 포트 바인딩]이라는 과정을 통해 별도로 연결됩니다. "사용되지 않음"이라고 뜨는 게 **100% 정상**입니다.

---

## 26. vCenter 백업 설정

### 백업 아키텍처

```
[ DNS 서버 (Win Server) ]
  └── C:\backup 폴더 생성
  └── IIS FTP 서버 설치 → FTP 사이트명: VCENTER-BACKUP
        └── 실제 경로: C:\backup
              ↓
[ vCenter 백업 설정 ]
  └── https://10.10.10.200:5480 → [백업] 메뉴
        └── FTP 주소: ftp://ftp.team1.com/backup
```

### DNS 서버에 FTP 서버 설치 (IIS)

```
서버 관리자 → 역할 및 기능 추가
  → 웹 서버(IIS) 선택 → 하위에서 [FTP 서버] 체크
    → 설치 완료

IIS 관리자 → [FTP 사이트 추가]
  → FTP 사이트 이름: VCENTER-BACKUP
  → 실제 경로(Physical path): C:\backup
  → [Next] → [Finish]
```

### vCenter 백업 설정 (VAMI)

```
https://10.10.10.200:5480 접속
  → [백업] 메뉴 → [설정] 탭
    → [편집] 클릭
      → 스케줄: 요일(금요일), 시간(9:25 UTC) 설정
      → 백업 위치: ftp://ftp.team1.com/backup
      → 데이터 백업 항목: Stats, Events, and Tasks / Inventory and configuration
      → 보존 백업 수: 모든 백업 보존
    → [저장]
```

### 백업 결과 확인

```
vCenter VAMI [백업] 탭 → [작업] 섹션
  → 최근 수행된 백업 항목 확인:
    백업 위치: ftp://ftp.team1.com/backup/vCenter/sn_vcsa.team1.com/M_7.0.3.02100_2026...
    버전: VC-7.0.3x
    사용자: administrator
    시작 시간: (날짜/시간 확인)

C:\backup 폴더에서 .gz 파일들 생성 확인:
  backup_label_file_data.tar.gz
  backup-metadata.json
  config_files.tar.gz
  database_full_backup.tar.gz
  full_wal_backup_meta.tar.gz
  lotus_backup.tar.gz
  vum.gz (약 179MB)
  ...등 다수 GZ 파일
```

---

## 27. DRS (Distributed Resource Scheduler)

> DRS는 클러스터 내 호스트들의 CPU·메모리 사용량을 분석하여 VM을 최적의 호스트로 배치·이동하는 기능입니다.

### 자동화 수준 3단계

#### 🔧 수동 (Manual)

DRS가 가상머신의 최적 배치를 **추천만 제공**하며, 실제 vMotion 마이그레이션은 관리자가 직접 수행해야 하는 방식입니다.

- 관리자가 모든 이동을 제어할 수 있지만 운영 부담이 커질 수 있습니다.

#### 🤝 부분 자동화 (Partially Automated)

가상머신이 **처음 전원이 켜질 때**는 DRS가 자동으로 호스트를 선택해 배치합니다.

- 이후 클러스터 부하 균형을 위한 마이그레이션은 추천만 제공하며 관리자가 승인해야 실행됩니다.

#### 🚀 완전 자동화 (Fully Automated)

VM 초기 배치와 이후 부하 균형을 위한 마이그레이션을 **모두 DRS가 자동으로 수행**합니다.

- 클러스터의 CPU·메모리 사용량을 분석해 최적의 호스트로 자동 이동합니다.

### DRS 설정 화면 구성

| 설정 항목 | 설명 |
|----------|------|
| **vSphere DRS** | 토글로 활성화/비활성화 |
| **자동화 수준** | 수동 / 부분 자동화 / 완전 자동화 선택 |
| **마이그레이션 임계값** | 일반(vMotion 빈도 낮음) ↔ 적극적(vMotion 빈도 높음) 슬라이더 |
| **Predictive DRS** | 예측 기반 선제적 VM 이동 (사용 체크 시 활성화) |
| **가상 시스템 자동화** | 개별 VM에 대한 자동화 예외 설정 |

### DRS 설정 경로

```
vCenter → 클러스터 선택 → [구성] → [vSphere DRS] → [편집]
  → 자동화 수준 선택
  → 마이그레이션 임계값 조정
  → [확인]
```

---

## 28. HA (High Availability)

> HA는 클러스터 내 호스트가 장애를 일으켰을 때, 해당 호스트에서 실행되던 VM들을 **다른 호스트에서 자동으로 재시작**하는 기능입니다.

### 고가용성 구조도

```
[ Cluster → DRS → HA ]
          │
    ┌─────┼─────┐
  vMotion  HA   FT
          │
[ 여러 ESXi 호스트들 ]
  각 호스트마다 VM들이 분산 실행 중
          │
  호스트 장애 발생!
          │
  [ HA가 감지 → 다른 호스트에서 VM 재시작 ]
```

### HA 실패 조건 및 응답 설정

| 실패 조건 | 기본 응답 |
|----------|----------|
| **호스트 장애** | VM 다시 시작 — VM 다시 시작 우선 순위를 사용하여 VM을 다시 시작합니다 |
| **호스트 격리** | 사용 안 함 — 포트로폴리오에서 VM본 전원 상태를 계속 유지합니다 |
| **Proactive HA** | 사용 안 함 — 데이터스토어 보호를 사용하도록 설정되었습니다. 정상 VM 다시 시작 시도합니다 |
| **PDU/네트워크에서 데이터스토어 손실이 있는 데이터스토어** | 전원을 끈 후 VM 다시 시작 — 데이터스토어를 보호하도록 설정하였습니다. VM이 시작되기 전까지 리소스를 시도합니다 |
| **VM 및 애플리케이션 모니터링은 사용하지 않도록 설정되어 있습니다** | — |

### Admission Control (승인 제어)

- HA가 VM들을 재시작할 수 있을 만큼의 리소스를 항상 **예약(Reserve)** 해두는 기능
- 방식: 퍼센트(%), 슬롯(Slot), 전용 장애 조치 호스트 중 선택

### HA 실습 — 하나의 데이터스토어에 클러스터 생성

```
vCenter → 클러스터 선택 → [구성] → [vSphere HA] → [편집]
  → vSphere HA 활성화 토글 ON
  → 실패 조건 및 응답 설정
  → [확인]

실험 방법:
  하나의 iSCSI 데이터스토어에 클러스터를 생성하고,
  특정 호스트를 강제로 다운시켜 VM이 다른 호스트에서 재시작되는지 확인
```

---

## 29. Resource Pool 설정 및 활용

> Resource Pool은 클러스터 또는 호스트 내에서 CPU·메모리 자원을 논리적으로 분할·제한·보장하는 기능입니다.

### Resource Pool 개념

```
[ 클러스터 전체 자원 ]
  CPU: 100% / Memory: 100%
        │
  ┌─────┴──────┐
  │ 리소스 풀 A│  → CPU Limit: 40% / Memory Limit: 40%
  │  (팀A용)  │
  └─────────────┘
        │
  ┌─────┴──────┐
  │ 리소스 풀 B│  → CPU Limit: 60% / Memory Limit: 60%
  │  (팀B용)  │
  └─────────────┘
```

### CPU 리소스 설정 항목

| 항목 | 설명 |
|------|------|
| **Shares (공유)** | 여러 풀이 경합할 때 우선순위 비율 (High/Normal/Low 또는 수동 입력) |
| **Reservation (예약)** | 이 풀에 항상 보장되는 최소 CPU/메모리 자원 |
| **Limit (제한)** | 이 풀에서 사용할 수 있는 최대 CPU/메모리 자원 |

### 실습 설정 예시 (Jongyeon-pool)

```
CPU 리소스 설정:
  Limit (제한): 1 GHz
    → 리소스 풀에서 사용할 수 있는 CPU 최대 사용량을 1GHz로 제한
    → VM들이 CPU를 요청하더라도 1GHz 이상은 사용할 수 없음
    → 여러 VM이 실행되더라도 리소스 풀 전체 CPU 사용량은 1GHz를 넘지 않음
```

리소스 풀 활용률 확인 경로:

```
vCenter → 리소스 풀 선택 → [모니터] 탭 → [활용률]
  → CPU 활용률 그래프 (0 kHz ~ 177 MHz 범위)
  → 메모리 활용률 그래프
  → 게스트 메모리 활용률
```

### Resource Pool 생성 절차

```
vCenter → 클러스터 또는 호스트 우클릭
  → [새 리소스 풀 (New Resource Pool)]
    → 이름 입력 (예: Jongyeon-pool)
    → CPU Shares / Reservation / Limit 설정
    → Memory Shares / Reservation / Limit 설정
    → [확인]
```

---

## 30. Linux 마스터 템플릿 제작 및 배포

> 여러 명의 개발자가 협업하는 환경에서 동일한 설정의 VM을 빠르게 배포하기 위해 **일반화(Generalization)** 과정을 거친 마스터 템플릿이 필수입니다.

### 왜 일반화 과정이 필요한가?

단순히 VM을 설치하고 복제하면, 원본 서버의 고유 정보(IP 주소, 하드웨어 주소, 고유 식별자)까지 그대로 복사됩니다. 이 과정을 건너뛰면 다음과 같은 문제가 발생합니다.

| 문제 | 원인 |
|------|------|
| IP 충돌 | 복제된 VM이 원본과 같은 IP를 가짐 |
| 호스트네임 충돌 | 여러 VM이 동일한 hostname 사용 |
| 머신 ID 중복 | DHCP, DNS 등에서 인식 오류 발생 |

마스터 템플릿은 이러한 '고유 정보'를 제거하여, 새로운 VM으로 태어날 때 vSphere로부터 새로운 정보를 주입받을 수 있는 **'깨끗한 도화지'** 상태를 만드는 작업입니다.

### 1단계: 사전 준비 — 필수 패키지 설치

vSphere가 외부에서 리눅스 내부의 호스트네임과 IP를 자동으로 수정하려면, OS 내부에 명령을 전달받을 에이전트(Agent)가 설치되어 있어야 합니다.

```bash
# vSphere와 통신하기 위한 필수 도구 설치
sudo dnf install open-vm-tools perl -y

# 서비스 활성화 및 시작
sudo systemctl enable vmtoolsd
sudo systemctl start vmtoolsd
```

### 2단계: 일반화 (Cleaning Up)

가장 중요한 단계입니다. 템플릿을 굽기 직전에 아래 명령어들을 순서대로 실행합니다.

**① 네트워크 설정 초기화**

특정 IP 주소 정보가 파일에 남아있으면 배포 시 충돌의 원인이 됩니다.

```
대상 파일: /etc/sysconfig/network-scripts/ifcfg-ens192 (또는 해당 장치명)

작업: IPADDR, NETMASK, GATEWAY, DNS, UUID, HWADDR 줄을 삭제하거나 주석 처리

최종 상태: BOOTPROTO=none (또는 static), ONBOOT=yes 만 남깁니다.
```

**② 시스템 고유 식별자(Machine ID) 제거**

모든 VM이 서로 다른 고유 ID를 가질 수 있도록 기존 ID를 지웁니다.

```bash
# Machine ID 초기화
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
```

**③ 네트워크 카드 룰 및 로그 청소**

복제 후 네트워크 인터페이스 이름이 밀리는 현상을 방지하고, 불필요한 로그를 지웁니다.

```bash
# 네트워크 룰 삭제
sudo rm -f /etc/udev/rules.d/70-persistent-net.rules

# 로그 및 임시 파일 삭제
sudo find /var/log -type f -exec truncate -s 0 {} \;
sudo rm -rf /tmp/*
sudo rm -rf /var/tmp/*
```

### 3단계: 기록 삭제 및 전원 종료

```bash
# 쉘 입력 기록 삭제 후 종료
history -c && history -w && sudo poweroff
```

### 4단계: 템플릿 변환 및 배포

```
1. 템플릿 변환:
   vSphere Client에서 해당 VM 우클릭
     → [템플릿] → [템플릿으로 변환]

2. 배포:
   생성된 템플릿 우클릭
     → [이 템플릿에서 새 가상 머신 생성]

3. 사용자 지정:
   '운영 체제 사용자 지정' 단계에서
     → 미리 만들어둔 Customization Spec 적용
     → 각자 원하는 IP와 호스트네임 입력
```

### 5단계: 배포 후 최종 확인

새로 만든 VM을 켜고 로그인한 뒤, 아래 명령어로 설정이 올바른지 검수합니다.

```bash
# vSphere에서 지정한 이름으로 바뀌었는가?
hostname

# 입력한 수동 IP가 정상적으로 할당되었는가?
ip a

# 외부 인터넷 통신이 가능한가?
ping 8.8.8.8
```

> 💡 **팀원 공지 팁**: 템플릿의 초기 계정 정보(ID/PW)를 팀원들에게 공유하고, 로그인 직후 `passwd` 명령어를 통해 개별 비밀번호로 변경하도록 안내하세요.

---

## 📁 프로젝트 주요 체크리스트

- [x] VMware Workstation 네트워크(VMnet) 구성 (VMnet0 Bridged, VMnet1 Host-only)
- [x] ESXi VM 생성 — CPU VT-x/EPT 중첩 가상화 옵션 활성화
- [x] ESXi 7.0 설치 (DCUI에서 Static IP, Subnet 설정)
- [x] DNS 서버 설정 및 호스트명(FQDN) 등록
- [x] vCenter에 전체 ESXi 호스트(12대) 등록
- [x] 데이터센터(DC) 생성 및 팀원별 구성
- [x] ESXi 사용자 계정 생성 + 관리자 권한 부여
- [x] 잠금 모드(Normal/Strict) 설정 및 예외 사용자 실습 검증
- [x] Shared Datastore (NFS/VMFS iSCSI) 개념 정리
- [x] Aruba Switch 기반 물리 네트워크 토폴로지 구성
- [x] 가상화 인프라 구성 요소 역할 이해 (ESXi / DNS / Storage / vCenter)
- [x] TrueNAS 스토리지 VM 사양 설정 (RAID-5, ZVOL/Dataset 구분)
- [x] ESXi 호스트별 VMkernel / vSwitch 설정 (Storage / vMotion 분리)
- [x] ZFS / ZVOL 개념 정리 (TrueNAS 스토리지 구조 이해)
- [x] 데이터스토어 자동 전파 원리 이해 (VMFS 클러스터 파일 시스템)
- [x] vCenter SSH 활성화 (VAMI 포털 통해 설정)
- [x] 오픈 소스 스토리지 솔루션 비교 (TrueNAS / OMV / StarWind)
- [x] TrueNAS 설치 및 초기 설정 (VM 생성 → ISO 설치 → 웹 UI 접속)
- [x] TrueNAS ZFS 계층 구조 이해 (Disk → VDEV → Pool)
- [x] TrueNAS RAID-Z1 Pool 생성 (Width 4 / VDEVs 1 / RAIDZ1)
- [x] TrueNAS VDEV 옵션 상세 이해 (Log / Spare / Cache / Metadata / Dedup)
- [x] Windows Server NTP 서버 설정 (레지스트리 수정 + w32tm 서비스 재시작)
- [x] Windows Server iSCSI Target 역할 추가 및 가상 디스크(.vhdx) 생성
- [x] IQN 개념 이해 및 ESXi IQN 등록 (Target 접근 제어)
- [x] ESXi iSCSI Initiator 설정 및 동적 검색 구성
- [x] VMFS 데이터스토어 생성 5단계 완료 (VMFS 6)
- [x] iSCSI 동적 검색 대상 제거 절차 학습 (정적 검색 → 동적 검색 순서)
- [x] ESXi 가상 네트워크 계층 구조 이해 (VMkernel → vSwitch → vmnic)
- [x] VLAN ID 개념 및 Untagged/Tagged 설정 이해
- [x] vCenter 백업 설정 (FTP 서버 + VAMI 백업 스케줄 구성)
- [x] DRS 설정 (수동 / 부분 자동화 / 완전 자동화 3단계 이해)
- [x] HA 설정 및 고가용성 실험 (단일 데이터스토어 클러스터)
- [x] Resource Pool 설정 및 CPU Limit 적용 (Jongyeon-pool)
- [x] Rocky Linux 마스터 템플릿 제작 및 배포 (일반화 → 템플릿 변환 → Customization Spec)

---

## 🔑 공통 자격증명 (실습 환경 전용)

| 구분 | 계정 | 비밀번호 |
|------|------|---------|
| ESXi root | `root` | `VMware1!` |
| vCenter 관리자 | `administrator@team1.vsphere` | `VMware1!` |
| TrueNAS 관리자 | `truenas_admin` | (설치 시 설정) |
| 팀원 계정 예시 | `jy` | `VMware1!` |

---

## 📚 학습 키워드

`VMware` `ESXi` `vCenter` `vSphere` `Nested Virtualization` `중첩 가상화`
`DCUI` `Lockdown Mode` `잠금 모드` `예외 사용자` `Exception User`
`NFS` `iSCSI` `VMFS` `SMB` `LUN` `TrueNAS` `ZFS` `ZVOL` `Dataset`
`RAID-Z1` `RAIDZ1` `VDEV` `Pool` `Width` `Log` `Spare` `Cache` `Metadata` `Dedup` `L2ARC` `ZIL`
`vMotion` `Shared Datastore` `DNS` `NTP` `VAMI` `SSH`
`Aruba Switch` `VLAN` `VMkernel` `vSwitch` `vmnic` `Port Binding`
`VMware Workstation` `Bridged Network` `Host-only Network`
`RAID-5` `Parity` `Self-healing` `Virtual Lab` `Sandbox`
`DRS` `HA` `High Availability` `Resource Pool` `Admission Control`
`vCenter Backup` `FTP` `IIS` `VAMI Backup`
`IQN` `iSCSI Target` `iSCSI Initiator` `Dynamic Discovery` `Static Discovery`
`Linux Template` `Generalization` `open-vm-tools` `Machine ID` `Customization Spec`
`OpenMediaVault` `OMV` `StarWind` `Fusion Pool` `Deduplication`
`Rocky Linux` `vmtoolsd` `ifcfg` `BOOTPROTO` `ONBOOT`
`Windows Server` `W32Time` `NtpServer` `AnnounceFlags` `w32tm`

---

## 🗒️ 참고 사항

- 본 실습은 **평가판 라이선스** 기준이며, 60일 후 만료됩니다.
- 모든 IP는 **수동(Static)** 으로 설정하며, DHCP는 비활성화합니다.
- vCenter 연동을 위해 각 ESXi의 **호스트명(FQDN)이 DNS에 등록**되어야 합니다.
- NIC 1(Management)에만 게이트웨이를 설정하고, NIC 2/3/4는 **게이트웨이 없음**으로 설정합니다.
- NFS는 각 ESXi 호스트마다 **개별 마운트 등록**이 필요합니다. (vCenter의 '모든 호스트에 마운트' 기능 활용 가능)
- iSCSI는 포맷 후 VMFS 표식이 디스크 헤더에 새겨지므로 **나머지 호스트가 자동 인식**합니다.
- vCenter SSH 접속 후 `shell` 명령어로 **Bash Shell 전환** 필요합니다.
- TrueNAS에서 NFS 공유는 **Dataset**, iSCSI 공유는 **ZVOL**을 사용합니다.
- TrueNAS RAID-Z1 구성 시 실습 환경에서는 Log/Spare/Cache/Metadata/Dedup 모두 Skip 권장합니다.
- iSCSI 동적 검색 대상 제거 시 **정적 검색 항목을 먼저 제거한 후** 동적 검색 항목을 제거해야 합니다.
- Linux 템플릿 제작 시 Machine ID, 네트워크 설정, 로그를 반드시 초기화해야 복제 후 IP/호스트네임 충돌이 발생하지 않습니다.
- vCenter 백업은 VAMI(`https://vCenter_IP:5480`) → [백업] 메뉴에서 FTP/FTPS/HTTP/HTTPS/NFS/SMB 등 다양한 프로토콜로 설정 가능합니다.
- DRS는 클러스터 단위 기능이므로 반드시 **클러스터 생성 후 활성화**해야 합니다.
