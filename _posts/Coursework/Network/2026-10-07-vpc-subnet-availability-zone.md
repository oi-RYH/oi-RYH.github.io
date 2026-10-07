---
title: "[네트워크] VPC와 서브넷 설계"
date: 2026-10-07 00:03:00 +0900
published: true
author: oi-RYH
categories: [Network]
tags: [네트워크, VPC, 서브넷, 가용 영역, NAT Gateway, AWS]
toc: true
toc_sticky: true
excerpt: "클라우드 안에 섬 하나를 설계한다면 주소와 출입구, 작업장과 금고를 어떻게 배치할까. Public·Private 서브넷과 가용 영역, 중복 없는 주소 계획을 정리한다."
---

> 이 글과 내용은 개인 공부를 위해 GPT와 Claude를 사용해서 작성되었습니다.

<video controls playsinline preload="metadata" aria-label="VPC와 서브넷 설계 설명 영상" style="display: block; width: 100%; max-width: 420px; margin: 1.5rem auto;">
  <source src="{{ '/assets/videos/network/vpc-subnet-availability-zone.mp4' | relative_url }}" type="video/mp4">
  <a href="{{ '/assets/videos/network/vpc-subnet-availability-zone.mp4' | relative_url }}">영상 파일 열기</a>
</video>

클라우드에 가게 하나를 올리는 데서 더 나아가, 여러 서버가 사용할 주소와 통신 경로를 직접 설계하고 싶어졌다. 이번에는 **섬 전체를 VPC**, 섬 안의 구역을 서브넷에 비유한다.

VPC는 물리적인 섬을 구입하는 것이 아니라 클라우드 안에 논리적으로 분리된 가상 네트워크를 구성하는 것이다. 여기서는 AWS의 IPv4 네트워크를 예로 든다.

## 섬 전체의 주소 범위

VPC의 주소 범위를 `10.0.0.0/16`으로 정하면 전체 주소 수는 다음과 같다.

```text
2^(32 − 16) = 65,536개
```

이 범위 안에서 서로 겹치지 않는 서브넷을 만든다. 전체 주소 수가 곧 배치할 수 있는 서버 수는 아니다. 일반적인 AWS IPv4 서브넷에서는 다섯 주소가 예약되므로 `/24`의 256개 중 보통 251개를 사용할 수 있다. 서비스가 여러 주소를 사용하기도 한다.

## 선착장과 Public 서브넷

**Internet Gateway**, 줄여서 IGW는 VPC와 인터넷을 잇는 출입구다. 선착장 앞의 광장은 Public 서브넷에 비유한다.

중요한 기준은 물리적인 위치가 아니라 **라우팅 테이블에 IGW로 향하는 직접 경로가 있는가**다. Public 서브넷에 서버를 놓았다고 무조건 인터넷에 공개되지는 않는다. IPv4로 직접 통신하는 인스턴스에는 공인 IPv4 주소가 필요하며 보안 규칙도 허용해야 한다. [AWS 인터넷 게이트웨이 문서](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html)

광장에는 외부 요청을 여러 서버로 나누는 인터넷용 로드 밸런서, 관리자 접속을 중계하는 바스천, 인터넷행 주소 변환을 담당하는 public NAT Gateway 등을 배치할 수 있다. 모든 설계에 세 시설이 전부 필수인 것은 아니다.

## 안쪽 작업장과 Private 서브넷

Private 서브넷은 IGW로 직접 향하는 경로가 없는 서브넷이다. 작업장에는 API나 GPU 서버, 금고에는 데이터베이스나 캐시를 둘 수 있다.

외부 요청이 공개 로드 밸런서를 통해 내부 서버에 전달되도록 구성할 수 있다. 이는 인터넷 사용자가 내부 서버에 직접 연결하는 것과 다르다. Private이라는 이름만으로 내부 서버 간 통신까지 자동 차단되는 것도 아니다. [AWS 서브넷 종류](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html)

## 서로 다른 기본 경로

영상에서는 다음과 같이 경로를 구성한다.

| 구역 | VPC 내부 목적지 | 그 밖의 IPv4 목적지 |
|---|---|---|
| Public | 10.0.0.0/16 → local | 0.0.0.0/0 → IGW |
| 외부 접속이 필요한 Private | 10.0.0.0/16 → local | 0.0.0.0/0 → NAT Gateway |
| 인터넷 접속이 불필요한 금고 | 10.0.0.0/16 → local | 인터넷용 기본 경로 생략 가능 |

Private 서버가 업데이트를 받는다면 `서버 → public NAT Gateway → IGW → 인터넷` 순서로 나간다. 서버가 먼저 시작한 요청의 답장은 돌아온다. “출항 전용 배”는 응답까지 막는다는 뜻이 아니라, 외부에서 먼저 시작한 새 연결을 이 경로로 내부 서버에 전달하지 않는다는 비유다.

VPC 내부 목적지는 local 경로로 전달하므로 Private의 모든 통신이 NAT를 거치는 것은 아니다.

## VPC와 가용 영역의 범위

가용 영역은 **Availability Zone**, 줄여서 AZ라고 한다. 비유에서는 섬 안의 서로 떨어진 두 지역이다. VPC는 여러 AZ에 걸칠 수 있지만, **서브넷 하나는 한 AZ 안에만 존재한다.** [AWS 서브넷과 가용 영역](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html#subnet-basics)

작업장 구역을 두 지역에 걸쳐 늘리는 것이 아니라, 왼쪽 지역에 서브넷 하나, 오른쪽 지역에 별도 서브넷 하나를 만든다. 그 안에 같은 역할의 서버를 배치하는 것이다.

여러 AZ에 서브넷만 만들어 놓는다고 서비스가 자동으로 복구되지는 않는다. 요청을 정상 서버로 분배하고, 필요한 데이터를 복제하며, 장애가 났을 때 다른 쪽으로 전환하는 구성이 함께 있어야 한다.

영상의 AZ별 NAT Gateway 구성도 같은 취지다. 한 AZ의 NAT에만 의존하면 그쪽 장애가 다른 AZ의 외부 접속에 영향을 줄 수 있으므로, 이 설계에서는 각 AZ에 NAT를 두고 같은 AZ의 서버가 사용하도록 한다. [AWS NAT Gateway 가용성 설명](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-basics.html)

## 역할이 읽히는 주소 계획

영상의 설계도는 모든 구역을 `/24`로 나누고 같은 역할을 양쪽에 배치한다.

| 역할 | AZ-a | AZ-c |
|---|---|---|
| Public | 10.0.1.0/24 | 10.0.2.0/24 |
| Private 작업장 | 10.0.11.0/24 | 10.0.12.0/24 |
| Private 금고 | 10.0.21.0/24 | 10.0.22.0/24 |

영상에서 ‘11번 구역’이라고 한 것은 **이 주소 계획에서 사용하는 세 번째 숫자**다. `10.0.11.0/24` 전체를 IP 주소 `11`이라고 부르는 것도 아니고, 모든 네트워크에서 세 번째 숫자가 언제나 구역 번호인 것도 아니다.

이 예에서는 `10.0`을 공통으로 두고 `/24` 단위로 나눴기 때문에 세 번째 숫자로 구역을 구별할 수 있다. 1·2는 Public, 11·12는 작업장, 21·22는 금고로 쓰기로 **우리가 정한 규칙**이지 AWS가 강제하는 역할 번호가 아니다.

## 주소 중복과 확장 여유

다른 VPC나 사내망과 연결할 계획이 있다면 상대 주소 범위도 확인한다. 겹치는 IPv4 CIDR을 가진 VPC끼리는 VPC Peering을 만들 수 없다. [AWS VPC Peering 제약](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html)

VPN도 주소가 겹치면 목적지 구분이 어려워지고 재주소화나 주소 변환 같은 추가 설계가 필요할 수 있다. 당장 우리 VPC 안에서만 겹치지 않는다고 끝나는 문제가 아니다.

처음부터 모든 범위를 촘촘하게 사용하지 않고, 새 서비스나 AZ를 추가할 여유도 남겨 둔다. 반대로 실제 필요를 무시한 지나치게 큰 주소 할당은 이후 다른 망과 연결할 때 제약이 될 수 있다.

## 구역 분할과 실제 보호

서브넷은 주소 범위를 나누고 경로와 보안 정책을 구분하는 단위다. 울타리만 그었다고 통신이 차단되거나 장애가 격리되는 것은 아니다.

보안 그룹과 네트워크 ACL 같은 규칙은 통신 범위를 통제한다. 여러 AZ 배치와 복제·장애 전환은 장애에 대비한다. **주소 계획, 경로, 보안 정책, 가용성을 함께 설계해야** 비유 속 섬이 실제로 운영 가능한 네트워크가 된다.

## 핵심 정리

- VPC는 가상 네트워크 전체이고, 서브넷은 그 안의 주소 구역이다.
- Public과 Private은 IGW로의 직접 경로 여부로 구분한다.
- NAT를 통해 내부에서 시작한 외부 연결의 답장은 돌아올 수 있다.
- 한 서브넷은 하나의 AZ에만 속하며, 이중화에는 실제 서버와 데이터 구성도 필요하다.
- 연결할 망과 주소 중복을 피하고 확장 여유를 남긴다.

이전 글: [방화벽과 외부 접속]({% post_url /Coursework/Network/2026-10-07-firewall-external-access %})
