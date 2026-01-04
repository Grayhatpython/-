# Multi-ISP + Campus VLAN Lab (EVE-NG)
신입 네트워크 엔지니어 포트폴리오 — 설계/구현/트러블슈팅 기록

EVE-NG 환경에서 **멀티 ISP(Primary/Backup) + 캠퍼스 VLAN 분리 + DHCP + NAT + (선택)BGP**를 하나의 토폴로지로 묶어 구현했습니다.  
“설정 나열”보다 **왜 이렇게 설계했는지**, 그리고 **장애가 났을 때 어떤 순서로 확인/수정했는지**를 문서로 남기는 것을 목표로 했습니다.

---

## 1) Topology
<img width="1339" height="1216" alt="스크린샷 2026-01-05 062954" src="https://github.com/user-attachments/assets/0638ed5a-f4df-4ab2-9294-9e83c534bcbe" />


### 계층 구성(단순화)
- **Core 계층 별도 구성 없음**: Core/Distribution을 분리한 3-Tier 풀스케일 구조가 아니라, 학습/검증 범위를 고려해 **Edge ↔ Distribution/Gateway ↔ Access**로 단순화했습니다.

---

## 2) 목표(What I built)
### 핵심 기능
- **VLAN 분리 + SVI 기반 Inter-VLAN Routing**
  - VLAN 10: 192.168.1.0/24 (User)
  - VLAN 20: 192.168.2.0/24 (User)
  - VLAN 99: 192.168.0.0/28 (Management)
  - VLAN 100: 100.1.1.32/28 (Server/Public Segment)
  - Distribution에서 SVI를 통해 VLAN 간 라우팅 수행

- **EtherChannel (L2 Trunk)**
  - Dist ↔ Dist 구간을 Port-Channel로 묶어 안정성/대역폭 확보

- **STP VLAN별 Root 설계**
  - VLAN10, VLAN20 각각 Root Primary/Secondary 또는 Priority로 루트 분리(로드 분산 가능)

- **HSRP(게이트웨이 이중화)**
  - VLAN10/20의 Default Gateway를 VIP(가상 IP) 로 통일
  - 장애 시 Gateway가 끊기지 않도록 구성

- **OSPF(내부 라우팅)**
  - Dist, Edge 내부 라우팅 활성화
  - SVI는 Passive로 전환하여 불필요한 OSPF 인접 형성/ECMP 혼선 방지
  - Dist-Dist 링크가 L2라 OSPF neighbor 생기지 않음
  
- **DHCP 운영**
  - DHCP 서버: 192.168.2.100(VLAN 20)
  - 사용자 단말은 DHCP로 IP 수신(필요 시 `ip helper-address`로 릴레이 구성)
  - *네트워크 장비(스위치/라우터)는 운영 안정성을 위해 기본적으로 Static을 전제로 하고, 실습 편의상 필요한 구간만 DHCP를 사용했습니다.*

- **멀티 ISP(Primary/Backup) 기본 구성**
  - ISP1(Primary) / ISP2(Backup)로 인터넷 업링크 이중화
  - 장애 전환 시나리오: 링크 다운/업, 경로 전환, NAT 동작 검증

- **NAT/PAT**
  - 일반 사용자(사설 대역)가 인터넷 접속 가능하도록 Edge에서 PAT 적용
  - 멀티 ISP 환경에서 출구 인터페이스 변화 시 NAT가 꼬이지 않도록 route-map 기반 NAT 분기 적용(아래 NAT 섹션 참고)

- **내부 서버 접근(원격 접속 시나리오 포함)**
  - VLAN 100(100.1.1.32/28)에서 내부 서버 운영(예: 100.1.1.33)
  - 내부 사용자 → 서버 접근/원격 접속(SSH/RDP 등) 시나리오를 염두에 두고 설계  
  - 외부 사용자 접근은 “공인 대역 광고/라우팅 + 보안 정책(ACL/방화벽)”을 전제로 확장 가능하도록 구성(Backlog 참고)

- **Management VLAN을 활용한 네트워크 장비 관리**
  - 스위치 장비마다 Management VLAN SVI에 관리 IP 할당
  - SSH를 통해서만 장비 접근 허용
  - 사용자 VLAN에서는 장비 관리 접근 불가

---

## 3) Addressing Plan
### Public IP Subnet (100.1.1.0/24)
| 구간 | 대역 | 설명 |
|---|---|---|
| ISP1 ↔ Edge | 100.1.1.0/28 | ISP1 회선 구간 |
| ISP2 ↔ Edge | 100.1.1.16/28 | ISP2 회선 구간 |
| Internal Public (Server) | 100.1.1.32/28 | 내부 서버용 “공인 세그먼트” |

### Campus VLAN
| VLAN | Subnet | 용도 |
|---|---|---|
| VLAN 10 | 192.168.1.0/24 | 사용자 |
| VLAN 20 | 192.168.2.0/24 | 사용자, DHCP Server(192.168.2.100) |
| VLAN 99 | 192.168.0.0/28 | 관리 |
| VLAN 100 | 100.1.1.32/28 | 서버(예: 100.1.1.33) |

### Edge ↔ Distribution L3 Links
- Edge ↔ DSW1: `10.1.1.0/30`
- Edge ↔ DSW2: `10.1.2.0/30`

> 참고: 서로 다른 물리 L3 링크는 **같은 서브넷을 공유하지 않도록** 구성했습니다. (ARP/라우팅 혼선 방지)

---

## 4) Routing / ISP 설계

### 기본 방향 (1차 버전)
- ISP1을 **기본 경로(Primary)** 로 사용
- ISP1 장애 발생 시 ISP2로 **자동 전환(Backup)** 되도록 설계

---

### 내부 라우팅 (IGP)
- Edge ↔ Distribution 구간에서는 내부 네트워크 대역
  - `192.168.1.0/24`
  - `192.168.2.0/24`
  - `100.1.1.32/28`
  를 **OSPF를 통해 동적으로 학습**하도록 구성했습니다.
- 사용자 VLAN 및 서버 대역은 Distribution에서 OSPF로 광고하고,
  ISP 방향 Default Route는 **Edge 라우터에서만 제어**하여
  내부 라우팅과 외부 경로 제어를 분리했습니다.

---

### ISP 장애 감지 및 자동 전환 (IP SLA + Object Tracking)
- 단순 Static Default Route는 링크가 살아 있어도
  실제 외부 통신이 불가능한 상황을 감지하지 못하는 한계가 있습니다.
- 이를 보완하기 위해 **IP SLA와 Object Tracking을 사용해**
  ISP1 경로의 실제 도달 가능성을 주기적으로 검사하도록 구성했습니다.

#### 동작 방식
- IP SLA를 통해 외부 목적지(예: 공용 DNS 또는 ISP 측 IP)에 ICMP Echo 수행
- Object Tracking으로 IP SLA 상태를 추적
- Tracking 상태에 따라 Default Route를 자동으로 활성/비활성화

ip sla 1
 icmp-echo 8.8.8.8 source-interface GigabitEthernet0/0
ip sla schedule 1 life forever start-time now

track 1 ip sla 1 reachability

ip route 0.0.0.0 0.0.0.0 100.1.1.1 track 1     ! ISP1 (Primary)
ip route 0.0.0.0 0.0.0.0 100.1.1.17 5          ! ISP2 (Backup)

---

## 5) BGP (선택 구성)
- Edge Router ASN: **100**
- ISP1/ISP2 ASN: **200**
- 100.1.1.0/24 대역을 “공인 IP 블록을 보유한 상황”으로 가정하여 구성
- 외부에서 내부 서버(VLAN 100)에 접근하는 시나리오를 확장할 수 있도록, 공인 세그먼트(100.1.1.32/28)를 분리해 두었습니다.

---

## 6) NAT Design (멀티 ISP에서 NAT 정합성 유지)
멀티 ISP에서 자주 터지는 문제는 라우팅과 NAT가 서로 다른 출구를 바라보는 경우입니다.

- 라우팅은 ISP2(백업)로 넘어갔는데,
- NAT가 ISP1 인터페이스만 기준이면
- 내부 사용자는 “인터넷이 그냥 끊긴 것처럼” 보입니다.

그래서 출구 인터페이스 기준으로 NAT를 분기했습니다(개념).

- 내부 트래픽 중
  - g0/1(ISP1)로 나가는 트래픽 → g0/1 주소로 PAT
  - g0/2(ISP2)로 나가는 트래픽 → g0/2 주소로 PAT

---

## 7) Switching / VLAN 운영 포인트
### 7-1. Trunk / Encapsulation
- 기본적으로 802.1Q(dot1q) 기준으로 구성
- Trunk는 “허용 VLAN(allowed vlan)”을 명확히 제한하는 방향으로 운영  
- 장비에 따라 `switchport trunk encapsulation dot1q`가 필요한 경우가 있지만, 최신/일부 플랫폼은 dot1q 고정이라 명령이 없을 수 있습니다.

### 7-2. MAC Address Table 활용(현장식 점검)
DHCP가 안 될 때, 라우팅부터 보기보다 먼저 아래를 확인했습니다.
- 해당 포트에 **MAC이 학습되는지**
- 올바른 VLAN으로 들어오는지(Access VLAN/Trunk Allowed VLAN)
- STP로 차단되는 포트가 있는지

관련 명령 예:
- `show mac address-table`
- `show interfaces trunk`
- `show spanning-tree`

---

## 8) 보안(Access 레벨에서 적용/계획)
### 8-1. Port Security (Access Switch)
- 사용자 단말 포트에 대해 MAC 학습/제한을 걸어 “의도치 않은 단말 변경”을 탐지/차단하는 구성을 고려했습니다.
- 과도하게 빡세게 걸면 운영이 불편해질 수 있어, 랩에서는 최소 정책부터 적용하는 방식으로 접근했습니다.

### 8-2. VLAN Hopping 대응(방어 설정)
“VLAN hopping 설정”은 공격이 아니라 **방어(완화) 설정**으로 정리했습니다.
- Access 포트는 고정: `switchport mode access`
- DTP 비활성(스위치 스푸핑 완화): `switchport nonegotiate`
- Native VLAN 정책(더블태깅 완화): Native VLAN 변경 및 필요 시 태깅 정책 검토
- Trunk는 허용 VLAN 최소화: `switchport trunk allowed vlan ...`

---

## 9) 모니터링/분석(추가 예정)
### 9-1. SPAN(스위치 포트 분석기)로 트래픽 미러링
- 특정 포트의 송수신 트래픽을 복제해 분석 포트로 보내고(미러링),
- 서버에서 pcap로 저장/분석할 수 있도록 확장 예정입니다.

### 9-2. Syslog 중앙 수집
- 모든 네트워크 장비(스위치/라우터)의 로그를 Syslog 서버로 모아
  장애 시점/원인 추적을 쉽게 만드는 구성을 추가 예정입니다.

---

## 10) 트러블슈팅 로그 (문제 → 원인 → 해결 → 검증)
### (1) SVI가 `administratively down/down`에서 안 올라옴
**증상**
- `interface vlan 100` 생성 후 IP 할당했는데 down
- `no shutdown` 해도 변화 없음

**원인**
- SVI는 해당 VLAN에 **UP 상태 포트가 최소 1개** 있어야 up

**해결**
- VLAN에 실제 포트(Access) 할당 또는 Trunk에 VLAN 허용 및 링크 up

**검증**
- `show vlan brief`, `show interfaces status`, `show ip interface brief`

---

### (2) IP SLA / Track 기반 전환에서 “복구가 안 되는” 문제
**증상**
- 주 회선 복구 후에도 `show track` down 유지
- 특정 대상은 g0/2에서는 ping 되는데 g0/1에서는 실패

**원인(정리)**
- SLA는 source-interface를 지정해도 실제 egress/리턴 경로가 어긋나면 실패할 수 있음  
  (소스는 g0/1인데 라우팅/리턴이 g0/2로 돌아가거나, 반대로 리턴이 막히는 케이스)

**해결 방향**
- SLA 대상에 대한 **호스트 라우트**로 경로 정렬
- 타겟을 더 “확실한 왕복 경로”로 선택

---

### (3) 서버가 다른 서브넷인데 응답이 없던 문제
**증상**
- 서버(또는 DHCP/Syslog/분석 서버)가 다른 서브넷에 있을 때,
  같은 VLAN에서는 보이는데 외부(다른 VLAN/외부망)에서는 응답이 불안정/없음

**원인**
- 서버의 **Default Gateway 미설정/오설정**

**해결**
- 서버 GW를 해당 VLAN의 SVI(게이트웨이 IP)로 정확히 설정

**검증**
- 서버에서 `route print`/`ip route` 확인, 다른 VLAN에서 ping/접속 테스트

---

### (4) DHCP: 서버는 Lease를 잡았는데 클라이언트는 0.0.0.0
**증상**
<img width="710" height="394" alt="스크린샷 2026-01-05 025626" src="https://github.com/user-attachments/assets/433729a7-49e4-4d2c-ad5e-d9386cc4a8aa" />
- VPCS에서 show ip 결과: 0.0.0.0/0

<img width="749" height="227" alt="스크린샷 2026-01-05 025619" src="https://github.com/user-attachments/assets/11738ad9-9a5f-4f6d-b20e-1ac7a41cdee1" />
- DHCP 서버에서는 바인딩이 잡힘:
- show ip dhcp binding에 192.168.1.11 할당 기록 존재

<img width="715" height="422" alt="스크린샷 2026-01-05 025606" src="https://github.com/user-attachments/assets/7d5b2dd2-02fc-40f1-bf11-efd41b7ed9d9" />
- debug ip dhcp server packet/events에서
- DHCPDISCOVER received ... through relay 192.168.1.2
- Sending DHCPOFFER ... (192.168.1.11)
- unicasting BOOTREPLY ... to relay 192.168.1.2

<img width="908" height="747" alt="스크린샷 2026-01-05 025743" src="https://github.com/user-attachments/assets/e22a239e-0df9-471e-aa8d-7b252b58ca00" />

**원인**
<img width="825" height="250" alt="스크린샷 2026-01-05 034847" src="https://github.com/user-attachments/assets/ed96817e-6142-4ca9-b873-1ee9f4511630" />
- HSRP로 게이트웨이를 가상 IP(VIP) 로 바꿨는데,
- DHCP pool의 default-router가 예전 게이트웨이 값으로 남아 있음

**해결**
- DHCP pool의 기본 게이트웨이를 HSRP VIP로 수정

**검증**
- VPCS에서 DHCP 재요청 후 IP/GW 정상 수신

---

### (5) HSRP 상태 이상(Init/unknown)
**증상**
<img width="729" height="438" alt="스크린샷 2026-01-05 033131" src="https://github.com/user-attachments/assets/2263d320-2379-42bf-b854-696bbbe6b312" />
- show standby brief에서 VLAN별 상태가 Init / unknown으로 표기되며 active/standby 정상 수립되지 않음

**원인**
- 해당 VLAN의 SVI가 down/down (VLAN이 안 올라오면 HSRP Init)

**해결**
- 해당 VLAN SVI -> no shutdown

**검증**
- show standby brief에서 정상 수립 확인

---

### (5) OSPF 라우트가 VLAN10/20/100으로 “3갈래 ECMP”로 보임
**증상**
- show ip route에서 같은 목적지(예: 10.1.1.0/30)가
- via 192.168.2.2 (Vlan20)
- via 192.168.1.2 (Vlan10)
- via 100.1.1.40 (Vlan100)
처럼 여러 SVI를 next-hop으로 갖는 형태로 표시

**원인**
- 사용자 VLAN SVI에서 OSPF를 활성화하여, 해당 SVI들이 OSPF 이웃으로 인식되었고,
- 그 결과 동일 목적지에 대해 다수의 동일 비용 경로(ECMP)가 생성되었다.

**해결**
- SVI는 Passive로 전환하고, OSPF neighbor는 “진짜 L3 uplink”에서만 형성
- passive-interface vlan number

**검증**
- show ip route ospf
  
---

### (6) 외부(8.8.8.8) ping 실패 — “ICMP type 3 code 1 host unreachable
**증상**
<img width="670" height="432" alt="스크린샷 2026-01-05 041338" src="https://github.com/user-attachments/assets/c8570bd5-62d4-488e-b99e-2191bdf5b8be" />
- ping 8.8.8.8 결과 : Destination host unreachable가 192.168.2.2에서 반환

**원인**
- 패킷이 192.168.2.2(중간 L3 장비)까지는 갔지만 그 장비가 외부로 나갈 기본 경로(default route) 를 몰라서 드롭

**해결**
- L3 장비(Distribution)에 default route 추가

**검증**
- ping 8.8.8.8

---

### (7) SSH timeout
**증상**
- 관리 PC(라우터)에서 SSH 시도 시 : % Connection timed out; remote host not responding

**원인**
- Loopback 인터페이스는 관리용 식별 주소로 유용하지만, L2 Access 스위치에서 Loopback만 존재하는 경우
- 실제 패킷을 송수신할 L3 출구 인터페이스가 없어 SSH와 같은 관리 트래픽의 응답을 전송할 수 없다.
- 따라서 L2 스위치의 관리 접속을 위해서는 Management VLAN의 SVI가 필요하며, 네트워크 분리 시에는 default gateway까지 함께 고려해야 한다.

**해결**
- Management VLAN99을 만들고, Access 스위치에는 VLAN99 SVI(관리 IP) 를 부여
- (서로 다른 대역에서 접속한다면) Access 스위치에 ip default-gateway <VLAN99_VIP> 설정
- Trunk에 VLAN99 허용

**검증**
- ssh -l [user] [ip]

---

## Verification Snapshots (Command Evidence)
> 아래 스냅샷은 EVE-NG에서 실제로 동작 확인을 위해 캡처한 CLI 결과입니다.  
> 토폴로지 이미지는 “최종 설계 방향”을 기준으로 정리했으며, CLI 스냅샷은 **HSRP/FHRP 적용 전 단일 게이트웨이 버전**

### 1) Edge Routing Table (Default Route + 내부 경로 학습)
- Default route가 ISP1(Primary) 방향(100.1.1.14)으로 설정되어 있고  
- 내부 대역(192.168.1.0/24, 192.168.2.0/24, 100.1.1.32/28)이 동적 라우팅(표시상 OSPF)으로 학습되는 것을 확인했습니다.

<img width="884" height="449" alt="a" src="https://github.com/user-attachments/assets/cb5278ab-dfc6-4930-b07a-56f1d0107d9d" />


### 2) IP SLA / Track 상태 (Primary 링크 감시)
- `track 1`이 `IP SLA 1 reachability`를 추적하고 있고, 현재 Reachability가 Up 상태임을 확인했습니다.

<img width="897" height="465" alt="b" src="https://github.com/user-attachments/assets/0215a882-9d69-4836-b6b3-c9124565dbbb" />


### 3) eBGP 세션 상태 (ISP1/ISP2 모두 Established)
- Local AS 100(Edge), Remote AS 200(ISP1/ISP2) 구성
- 두 이웃(100.1.1.14 / 100.1.1.30) 모두 Up 상태로 BGP 세션이 형성됨을 확인했습니다.
- Prefix 수신(0)은 “최소 구성/필터링/Default-only 설계” 단계에서 의도적으로 단순화한 상태일 수 있습니다.

<img width="894" height="450" alt="c" src="https://github.com/user-attachments/assets/19d38194-4302-4e76-a818-4bb431f101bb" />


### 4) IP SLA 설정 값 (ICMP-Echo + Source Interface 고정)
- IP SLA 1이 `icmp-echo`로 8.8.8.8을 대상으로 테스트하며,
- Source interface를 GigabitEthernet0/1로 고정해 Primary 경로 유효성 검증에 활용했습니다.

<img width="897" height="464" alt="d" src="https://github.com/user-attachments/assets/aa8d4004-9259-4e34-9414-c6721b5c98af" />


### 5) NAT Statistics (Dual Outside + Route-map 기반 NAT 준비)
- Outside: Gi0/1(ISP1), Gi0/2(ISP2)
- Inside: Gi0/0(내부 방향)
- Route-map 기반으로 Primary/Back-up NAT 정책이 준비되어 있음을 확인했습니다.

<img width="927" height="494" alt="e" src="https://github.com/user-attachments/assets/990a1f91-ded1-4de0-9b0b-fd145c8407f1" />


---


## 11) Backlog (다음 확장 계획)
- **트래픽 분산(중요/일반 트래픽 분리)**
  - PBR로 소스/서비스 기반 출구 ISP 선택
  - BGP 정책(Local-Pref/AS-Path)로 인바운드/아웃바운드 제어
- **SPAN + 중앙 Syslog 수집 서버 구성**
- **Access Port Security 정책 고도화**
- **VLAN hopping 대응 설정을 템플릿화(Access/Trunk 공통 정책)**
- EVE-NG에 Windows/Linux 서버를 배포하여 내부 Public IP 기반 HTTP 서비스 접근 시나리오 구현

---

## 12) 참고 명령어 (검증에 사용)

### VLAN / SVI
- `show vlan brief`
- `show ip interface brief`
- `show interfaces status`
- `show interfaces vlan <vlan-id>`

### Trunk / EtherChannel
- `show interfaces trunk`
- `show etherchannel summary`
- `show etherchannel detail`

### MAC / L2 학습
- `show mac address-table`
- `show mac address-table vlan <vlan-id>`

### STP
- `show spanning-tree`
- `show spanning-tree vlan <vlan-id>`
- `show spanning-tree summary`

### OSPF
- `show ip ospf neighbor`
- `show ip ospf interface brief`
- `show ip route ospf`
- `show ip protocols`
- `clear ip ospf process`

### HSRP / Gateway 이중화
- `show standby brief`
- `show standby`

### DHCP
- `show ip dhcp binding`
- `show ip dhcp pool`
- `show run | section dhcp`
- `debug ip dhcp server events`
- `debug ip dhcp server packet`

### Routing / Default Route
- `show ip route`
- `show ip route 0.0.0.0`

### IP SLA / Object Tracking
- `show ip sla statistics`
- `show ip sla configuration 1`
- `show track`
- `show track 1`

### BGP
- `show ip bgp summary`
- `show ip bgp`
- `show ip bgp neighbors <x.x.x.x>`

### NAT
- `show ip nat translations`
- `show ip nat statistics`

### Management / SSH
- `show ip ssh`
- `show run | section line\ vty`
- `show users`
- `show access-lists`

### Interface / 장애 분석
- `show interfaces`
- `show ip interfaces brief`
- `show logging`
- `show run`

### Icmp
- `debug ip icmp`
---
