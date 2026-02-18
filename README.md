# Nangman Infra (Daejeon ↔ Seoul ↔ AWS) Network Overview

이 저장소는 대전(Daejeon) 온프레미스 네트워크와 서울(Seokchon) 온프레미스 네트워크를 출발점으로, AWS를 안전하게 연결한 하이브리드 인프라의 네트워크 설계를 문서화한다.  
모든 외부 요청은 서울의 OPNsense로 유입되고, 내부의 Nginx Proxy Manager가 도메인 기반으로 각 서비스로 트래픽을 전달한다.

## Topology Diagram

![Topology](./Topology.png)

## Address Plan (CIDR)

### 1) Daejeon LAN
- (기존) Private CIDR: `192.168.10.0/23`
- 문제: 서울의 `192.168.10.0/24` 와 대전의 `192.168.10.0/23` 이 겹쳐서, 일반적인 라우팅으로는 두 사이트를 그대로 붙일 수 없다(대역 중복).

### 2) Seoul (Seokchon) LAN
- Private CIDR: `192.168.10.0/24`
- 인바운드 시작 지점: OPNsense (서울 경계 방화벽)

### 3) WireGuard Overlay (대역 중복 해결용)
- WireGuard CIDR: `172.16.0.0/23`
- 목적: “대전 ↔ 서울”을 물리 LAN 대역이 아니라, 겹치지 않는 별도의 Overlay 대역으로 묶어서 충돌을 회피한다.
- 결과: 대전 서비스/서버는 서울에서 접근할 때 WireGuard 주소(172.16.x.x)를 기준으로 접근한다.

### 4) AWS VPC
- AWS VPC CIDR: `10.120.0.0/20`
- AWS에서 운영하는 서비스들은 이 VPC 내부에서 동작한다.

## Site-to-Site Connectivity

### A) Seoul ↔ AWS : IPsec Site-to-Site VPN
- 배경: 운영 서비스들을 AWS에서 돌리기로 결정했기 때문에, 서울 온프레미스와 AWS를 안전한 터널로 상시 연결한다.
- 방식: 서울 OPNsense를 기준으로 AWS와 IPsec Site-to-Site로 연결한다.
- 라우팅 관점:
  - 서울에서 `10.120.0.0/20`(AWS VPC)로 가는 트래픽은 IPsec 터널로 보낸다.

### B) Seoul ↔ Daejeon : WireGuard Site-to-Site (Overlay)
- 배경: 대전 LAN 대역이 서울 LAN과 겹치므로(LAN-to-LAN 직접 라우팅 불가), 겹치지 않는 새 주소 공간을 만들어 우회한다.
- 방식: WireGuard(예: `172.16.0.0/23`)로 양 사이트를 묶고, 대전 리소스는 WireGuard IP로 식별한다.
- 라우팅 관점:
  - 서울에서 `172.16.0.0/23`로 가는 트래픽은 WireGuard 인터페이스로 보낸다.

## Inbound Traffic Flow (요청이 흘러가는 순서)

외부 사용자의 모든 요청은 “서울”로 먼저 들어오고, 서울에서 도메인 기준으로 내부 서비스에 분기된다.

1) Users → (Public Internet)  
2) 서울 OPNsense (경계 방화벽)
   - 들어오는 트래픽을 1차로 필터링한다(허용 포트/차단 정책 적용).
3) 서울 Nginx Proxy Manager (Reverse Proxy)
   - 요청의 Host(도메인)를 보고, 설정된 Upstream으로 전달한다.
4) 목적지 서비스
   - 서울 내부 서비스면: 서울 LAN(192.168.10.0/24)로 전달
   - 대전 서비스면: WireGuard Overlay(172.16.0.0/23)로 넘어가서 전달
   - AWS 서비스면: IPsec 터널을 통해 AWS VPC(10.120.0.0/20)로 전달

정리하면,
- 보안/관문: 서울 OPNsense
- 라우팅/분기: 서울 Nginx Proxy Manager
- 원격 연결:
  - AWS는 IPsec
  - 대전은 WireGuard(대역 중복 회피)

## Routing Notes (구현할 때 보통 필요한 라우팅 개념)

- 서울 OPNsense 기준으로 다음 경로들이 “어느 터널로 나가야 하는지” 결정돼야 한다.
  - AWS VPC(`10.120.0.0/20`) → IPsec 터널로 라우팅
  - WireGuard Overlay(`172.16.0.0/23`) → WireGuard 인터페이스로 라우팅

- Nginx Proxy Manager는 도메인별로 Upstream을 잡을 때,
  - 서울 서비스: `192.168.10.x` 같은 서울 LAN IP로 지정하거나,
  - 대전 서비스: `172.16.x.x` 같은 WireGuard IP로 지정하거나,
  - AWS 서비스: `10.120.x.x` 같은 VPC IP(또는 내부 LB/DNS)로 지정한다.

## Why This Design (이 구조를 선택한 이유)

- 인바운드 단일화: 외부 트래픽이 서울로만 들어오게 해서 보안 정책과 운영 포인트를 단순화한다.
- 경계 보안 강화: OPNsense에서 1차 방어, Reverse Proxy에서 2차 분기 및 서비스 노출 제어가 가능하다.
- 대역 중복 문제 해결: 대전과 서울 LAN이 겹치므로, WireGuard Overlay 대역을 별도로 만들어 충돌 없이 연결한다.
- 클라우드 운영 전환: 운영 서비스는 AWS에서 구동하되, 서울과의 안전한 사설 통신이 필요하므로 IPsec으로 상시 연결한다.

