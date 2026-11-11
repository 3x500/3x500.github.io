---
layout: single
title: "AD 15. Site 물리 설계: Subnet·WAN·장애 도메인·통신 구간 검증"
categories: [Active Directory]
tags: [AD, Site, Subnet, DC Locator]
toc: true
published: false
---

같은 도메인을 사용하는데 지점 PC가 먼 지역의 DC와 통신할 수 있다. OU가 지점별로 나뉘었다고 가까운 DC가 자동으로 선택되는 것은 아니다. 객체 관리 구조와 네트워크 위치를 구분해야 한다.

Domain·OU가 업무 분류라면 Site·Subnet은 네트워크 지도에 가깝다. AD의 Subnet 객체를 Site와 연결해 클라이언트 위치 판단에 활용한다. Site Link는 사이트 사이 복제 경로와 비용·일정 등을 표현한다. [Microsoft Site·Subnet·Site Link](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/replication/active-directory-replication-concepts)

## 지점 이름만 만드는 것으로 끝나지 않는다

| 항목 | 확인할 내용 |
|---|---|
| Site | 네트워크 위치를 어떤 단위로 정의했는가? |
| Subnet 매핑 | 실제 클라이언트 주소 범위가 올바른 Site에 연결됐는가? |
| DC 배치 | 해당 Site에서 사용할 수 있는 DC가 있는가? |
| Site Link | 회선과 복제 비용·일정을 어떻게 반영했는가? |

새 지점이나 VPN 주소 대역을 추가했는데 Subnet 등록을 놓치면 위치 판단이 기대와 달라질 수 있다. 지점별 OU를 추가하는 작업과 별도 확인 항목으로 관리한다.

## 클라이언트 관점에서 조회한다

해당 지점의 도메인 멤버 PC에서 실행하는 진단 예제다. 실행 계정·도구 설치 상태에 따라 권한과 사용 가능 여부를 확인한다.

```text
nltest /dsgetsite
nltest /dsgetdc:ad.example.com
```

첫 결과에서 클라이언트 Site를, 두 번째에서 검색된 DC와 Site 관련 정보를 확인한다. 이는 DC 검색 결과이며, 해당 PC의 모든 애플리케이션이 그 DC를 사용한다는 의미는 아니다. 애플리케이션은 서버를 직접 지정하거나 연결을 캐시할 수도 있다. [Microsoft DC Locator](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/dc-locator), [Microsoft nltest](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/cc731935(v=ws.11))

## 원본 물리 설계 원칙을 배치표로 바꾼다

| 설계 축 | 결정할 것 | 도면과 실제 검증 |
|---|---|---|
| 사용자 위치 | 본사·지사·VPN 대역과 Site 매핑 | 대표 단말 주소와 `nltest /dsgetsite` |
| DC 배치 | 로컬 인증 요구·회선 단절·운영 책임 | DC Site·DNS/GC 역할·장애 시 대체 경로 |
| 장애 도메인 | 호스트·스토리지·랙·전원·회선 | 두 DC가 같은 장애에 함께 중단되는지 |
| 복제 | 링크 비용·스케줄·주기와 필요한 수렴 시간 | 실제 파트너·마지막 성공·WAN 사용량 |
| 관리·백업 | 관리 단말 경로·복구 저장소 접근 | 업무 질의와 관리 접근·백업 경로를 분리해 검사 |

같은 Site에 로컬 DC가 있어도 모든 앱이 WAN 의존성을 잃는 것은 아니다. 다른 서비스·도메인·GC·역할 보유자나 앱이 직접 지정한 서버까지 고려한다. Site Link의 비용은 경로 선택의 상대값으로 설계하며 실제 지연 밀리초와 같다고 설명하지 않는다. [Site 토폴로지 설계의 범위](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/designing-the-site-topology)

설명용으로 본사 `HQ`, 지사 `Branch`, VPN 대역을 정의한다. 지사 OU와 Site는 다른 대상이다. 신규 대역·VPN 라우팅 변경을 네트워크 작업으로만 끝내지 않고 Subnet 매핑과 대표 단말의 검색 결과까지 변경 기록에 포함한다.

## 포트 목록을 출발지·목적지·기능으로 나눈다

원본의 필수 AD Port 표는 방화벽 설계의 출발점이다. 실제 필요한 규칙은 역할·OS·프로토콜·정책과 양방향 요청 관계에 따라 정한다. 다음 대표 목록을 모든 장비에 일괄 허용하는 규칙으로 사용하지 않는다.

| 기능 | 대표 포트·프로토콜 | 시험 범위 |
|---|---|---|
| DNS | TCP/UDP 53 | 클라이언트→DNS, 전달 DNS→대상 DNS |
| Kerberos | TCP/UDP 88, 암호 변경 464 | 인증·암호 변경 경로 |
| LDAP·DC Locator | TCP/UDP 389 | 디렉터리 연결과 UDP LDAP 응답 구분 |
| SMB·SYSVOL | TCP 445 | 클라이언트의 정책 파일 접근 등 |
| GC·TLS LDAP | TCP 3268/3269, 636 | 실제 사용 서비스와 TLS 조건 |
| RPC | TCP 135와 실제 동적·고정 포트 구성 | DC 간 복제 등 구체 기능의 전체 경로 |
| 시간 | UDP 123 | 계층·외부 시간 원본별 허용 경로 |

현대 Windows의 동적 RPC 기본 범위는 보통 TCP 49152–65535이지만 고정·제한 설정과 OS 조건을 확인한다. Endpoint Mapper 135 성공만으로 후속 RPC 통신까지 검증한 것은 아니다. [AD 도메인·Trust 방화벽 구성](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/config-firewall-for-ad-domains-and-trusts)

## 가까운 DC를 못 쓰는 원인

로컬 DC의 상태·기능, 사이트별 DNS 레코드, 서브넷 매핑, 회선·방화벽을 함께 본다. 가까운 DC가 없거나 사용할 수 없으면 다른 위치의 DC를 선택할 수 있다. “항상 같은 지점 DC만 사용해야 정상”이라고 판단하지 않는다.

변경 후에는 관리 콘솔의 지도뿐 아니라 실제 지점 PC에서 검색 결과와 인증 지연을 확인한다. 관리자가 본사 PC에서 확인한 결과로 지점 동작을 대신할 수는 없다.

Subnet·Site Link 변경 전 매핑·비용·일정·DC 위치·대표 단말의 기준 결과를 기록한다. 예상 밖 원격 DC 선택이나 지연·복제 지체가 생기면 확대를 멈추고 이번 변경분을 이전 값으로 복원한 뒤 같은 지점에서 재검증한다. 캐시된 연결이 있어 콘솔 설정 변경과 앱 연결 전환이 동시에 일어난다고 가정하지 않는다.

---

적용 기준: Windows Server AD DS의 Site·Subnet·DC Locator. 버전별 DC 검색 차이는 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.

[연재 목차]({% post_url 2026-08-13-ad-series-guide %})
