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

## 💾 03 — 공유 스토리지 구성 (iSCSI / NFS)

> 모든 ESXi 호스트가 동일한 스토리지를 공유하는 환경 구성 — vMotion · HA의 필수 전제 조건

### 1. Windows Server iSCSI Target 설정

- Windows Server에서 iSCSI Target 역할 추가 및 가상 디스크(VHD) 생성
- 이니시에이터 IQN 등록 및 접근 권한 부여

### 2. ESXi iSCSI Initiator 연결 및 VMFS 데이터스토어 생성

- 각 ESXi 호스트에서 소프트웨어 iSCSI 어댑터 추가
- Storage 전용 VMkernel에 포트 바인딩하여 트래픽 격리
- Dynamic Discovery로 Target 등록 → LUN 인식 → VMFS 6 데이터스토어 생성

### 3. NFS 공유 스토리지

- NFS Share 마운트 → ESXi 전체 호스트 자동 전파
- 데이터스토어 자동 전파 원리 (vCenter 중앙 관리 / 동일 스토리지 구독 / 클러스터 내 공유 정책)

### iSCSI vs NFS 비교

| 항목 | iSCSI (VMFS) | NFS |
|------|-------------|-----|
| 프로토콜 계층 | 블록 스토리지 | 파일 스토리지 |
| 동시 접근 | 단일 호스트 전용 | 다중 호스트 동시 가능 |
| 주요 용도 | 고성능 VM 워크로드 | 공유 템플릿 · ISO 배포 |

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

# 🚀 05 — 서비스 배포 및 구성

> **NFS 공유 스토리지를 활용한 고가용성 웹서비스 구축** — 웹 콘텐츠를 단일 NFS 스토리지에서 관리하여 모든 웹 서버가 동일한 데이터를 서비스하도록 구성했습니다.

---

## 환경 구성

| 역할 | VM 이름 | IP |
|------|---------|-----|
| NFS 서버 | NFS-Ser | 10.10.40.160 |
| 웹 서버 1 | NFS-Web01 | 10.10.40.161 |
| 웹 서버 2 | NFS-Web02 | 10.10.40.162 |
| 웹 서버 3 | NFS-Web03 | 10.10.40.163 |

웹 서버 3대는 모두 NFS 서버의 /nfs/web 디렉터리를 /var/www/html에 마운트하여 **동일한 콘텐츠를 서비스**합니다.

---

## 1. OVF 템플릿 기반 VM 배포

사전에 ESXi/vCenter에서 내보내기한 **OVF 템플릿**을 기반으로 Linux VM을 배포했습니다.
OVF 파일을 vCenter에서 가져오기(Import)하여 NFS 서버 및 웹 서버 VM 4대를 생성했으며,
배포 후 각 VM의 IP와 호스트명을 개별 설정했습니다.

| 자동 주입 항목 | 처리 방식 |
|----------------|-----------|
| IP / 서브넷 / 게이트웨이 | 배포 후 각 VM에서 수동 설정 |
| 호스트명 | 배포 후 개별 설정 |
| VMware Tools | OVF 템플릿에 사전 포함 |

---

## 2. vSphere 가상 스위치 구성


관리 트래픽과 서비스 트래픽을 분리하여 안정성을 높였습니다.

| 스위치 | 포트 그룹 | 연결 VM |
|--------|-----------|---------|
| vSwitch1 | 관리망 (vmk0: 10.10.10.100) | ESXi 호스트 관리 |
| vSwitch1 | VM Network | NFS-Ser, NFS-Web01/02/03, DNS-Server 등 7대 |
| vSwitch1 | vMotion-Network | vMotion 전용 |
| vSwitch2 | Internet-Network | Win-Server |

---

## 3. NFS 서버 구성 (NFS-Ser / 10.10.40.160)

### 3-1. 웹 콘텐츠 준비

```bash
[root@localhost web]# ls /nfs/web/
index.html

[root@localhost web]# cat /nfs/web/index.html
<h1>hello2 team1.com</h1>
```


### 3-2. /etc/exports 설정

```bash
[root@localhost web]# cat /etc/exports
/nfs/web  *(rw,sync,no_root_squash)
```

![/etc/exports 설정 확인](./images/img_5_1.png)

| 옵션 | 설명 |
|------|------|
| rw | 읽기/쓰기 허용 |
| sync | 디스크 쓰기 완료 후 응답 (데이터 안전성) |
| no_root_squash | 클라이언트 root를 서버 root로 매핑 허용 |

### 3-3. NFS 서비스 적용 및 확인

```bash
# 익스포트 목록 갱신
exportfs -r

# NFS 서버 재시작
systemctl restart nfs-server

# 현재 익스포트 목록 확인
exportfs -v
/nfs/web  <world>(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)
```

![NFS exportfs 적용 확인](./images/img_5_2.png)

```bash
# 서비스 상태 확인
systemctl status nfs-server
# ● nfs-server.service — Active: active (exited)
```

![NFS 서버 서비스 상태](./images/img_5_3.png)

---

## 4. NFS 클라이언트 마운트 (웹 서버 VM 3대)

NFS-Web01(161), NFS-Web02(162), NFS-Web03(163) 각각에서 동일하게 마운트합니다.

```bash
# NFS 공유 마운트
mount -t nfs 10.10.40.160:/nfs/web /var/www/html

# 마운트 결과 확인
ls /var/www/html
# cgi-bin  html → html/

cd html/
cat index.html
# <h1>hello2 team1.com</h1>
```

![NFS 마운트 성공 및 콘텐츠 확인1](./images/img_5_4.png)


---

## 5. 트러블슈팅: NFS-Web03 Access Denied

NFS-Web03(163)에서 마운트 시 **`access denied`** 오류가 발생했습니다.

### 증상

```bash
mount 10.10.40.160:/nfs/web /var/www/html
# mount.nfs: access denied by server while mounting 10.10.40.160:/nfs/web
```

![Access Denied 오류](./images/img_5_5.png)

### 원인 분석 및 조치

| 항목 | 상태 | 조치 |
|------|------|------|
| SELinux | **Enforcing** | Permissive 모드로 전환 |
| firewalld | **Active (Running)** | 중지 및 비활성화 |

```bash
# SELinux 상태 확인 → Enforcing
getenforce
# Enforcing

# SELinux Permissive 모드로 전환
setenforce 0
getenforce
# Permissive

# firewalld 중지 및 비활성화
systemctl stop firewalld
systemctl disable firewalld

systemctl status firewalld
# Active: inactive (dead) — disabled
```

![SELinux 및 firewalld 조치](./images/img_5_6.png)

![firewalld 비활성화 확인](./images/img_5_7.png)

> **원인**: SELinux의 NFS 보안 정책이 마운트 요청을 차단하고 있었음.
> 운영 환경에서는 semanage fcontext로 SELinux 정책을 세밀하게 조정하는 것을 권장.

### 해결 — 재마운트 성공

```bash
mount -t nfs 10.10.40.160:/nfs/web /var/www/html
# 정상 마운트 완료
```

---

## 6. 웹 서비스 동작 확인

브라우저에서 각 웹 VM IP로 접속하여 동일한 콘텐츠가 서비스됨을 확인했습니다.

```
http://10.10.40.161  →  hello2 team1.com  ✅
http://10.10.40.162  →  hello2 team1.com  ✅
http://10.10.40.163  →  hello2 team1.com  ✅
```

![웹서비스 확인](./images/img_5_8.png)

> **핵심 검증**: NFS 공유 스토리지의 **index.html** 파일 하나를 수정하면 **모든 웹 VM에 즉시 반영**됩니다.

---

## 7. NTP 서버 동기화

Windows Server를 도메인 NTP 서버로 구성하고, 전체 ESXi 호스트의 시간을 동기화했습니다.

```powershell
# Windows Server NTP 서버 설정
w32tm /config /manualpeerlist:"time.windows.com" /syncfromflags:manual /reliable:YES /update
net stop w32tm && net start w32tm
```
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
