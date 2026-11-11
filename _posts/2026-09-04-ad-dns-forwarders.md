---
layout: single
title: "AD 06. DNS 전달 설계: 이름 공간·타임아웃·대체 경로를 검증하기"
categories: [Active Directory]
tags: [AD, DNS, Forwarder, Security]
toc: true
toc_sticky: true
published: true
---

조건부 전달자에 서버 주소를 입력하고 이름이 조회됐다고 해서 설계한 전달 경로가 검증된 것은 아니다. 캐시나 로컬 영역에서 응답했을 수도 있다. 이 글에서는 원본 DNS 운영 문서의 전달 설정·장애 시나리오를 **이름 공간의 소유자, 질의 출발점, 실패 응답, 원복 조건**으로 나눠 설명한다.

## 전달·위임·사본 중 무엇이 필요한가

설명용 환경은 `ad.example.com` 내부 DNS와 별도 이름 공간 `partner.example.net`을 사용한다. 상대 조직의 이름을 조회하는 요구와 상대 디렉터리에 접근 권한을 얻는 요구는 다르다. DNS 전달은 Trust나 리소스 권한을 자동으로 만들지 않는다.

| 방법 | 구성하는 것 | 선택 조건과 운영 책임 |
|---|---|---|
| 서버 수준 Forwarder | 일반 미해결 재귀 질의의 상위 DNS | 인터넷 등 일반 조회를 승인된 경로로 모음. 범위가 넓어 변경 영향도 큼 |
| Conditional Forwarder | 특정 접미사의 질의를 보낼 DNS 목록 | 별도 이름 공간을 특정 DNS에 문의. 주소·대체 경로·복제 범위 관리 |
| Delegation | 부모 영역에서 자식 영역의 권한 DNS를 안내 | 하위 이름 공간의 권한을 분리. 참조와 필요 시 glue 관리 |
| Stub zone | 권한 DNS를 찾는 최소 레코드와 갱신 경로 | 권한 서버 목록 추적이 목적. 전체 레코드 사본을 제공하지 않음 |
| Secondary zone | 영역 전송으로 받은 읽기 전용 레코드 사본 | 로컬에서 영역 데이터를 제공해야 함. 전송·갱신·만료 관리 |

전달은 질의 경로, Secondary는 데이터 사본이다. Stub 역시 전체 사본 이중화의 대체물이 아니다. [Microsoft DNS 영역 유형](https://learn.microsoft.com/en-us/windows-server/networking/dns/zone-types)

## 로컬 영역이 있는지부터 확인한다

다음은 일반 재귀 질의를 이해하기 위한 흐름이다. DNS 정책과 `ForwardDelegations` 같은 설정에 따라 세부 경로는 달라질 수 있다.

```text
Client → 내부 DNS의 권한 영역·캐시 확인
       → 일치하는 조건부 전달 경로 확인
       → 적용 가능한 위임 확인
       → 서버 수준 전달자
       → 허용된 경우에만 Root Hints 기반 반복 조회
```

중간에 유효한 최종 응답을 받으면 끝난다. 내부 DNS가 해당 이름 공간의 권한 영역을 호스팅하고 있고 요청 레코드가 없다면, 외부 DNS가 그 레코드를 알고 있을 것이라는 이유로 일반 전달자에 넘기지 않는다. 로컬 영역과 같은 이름의 조건부 전달자도 만들 수 없다. 부모·자식 이름 공간은 영역 소유 관계와 위임부터 검토한다. [Microsoft DNS 전달 경로](https://learn.microsoft.com/en-us/windows-server/networking/dns/forwarding#how-dns-server-selects-a-resolution-path)

내부와 외부에 같은 이름 공간을 쓰는 Split-Brain 설계에서는 내부 권한 영역이 외부의 정상 레코드를 가릴 수 있다. 원본 구축 문서의 내부·외부 질의 분리 요구를 적용할 때는 내부 영역의 레코드 유지, 자식 영역 위임, DNS 정책 사용 여부를 먼저 결정한다. 전달자를 추가하는 것만으로 누락 레코드가 채워지지는 않는다.

## 원본 설정 화면에서 보이는 범위

![조건부 전달자 입력 화면. DNS 이름과 전달 대상 주소는 검정색으로 가림]({{ '/images/ad-deep-dive/b-079.png' | relative_url }})

원본에는 대상 이름과 서버 주소를 입력하는 영역이 있고, AD 저장 체크는 선택되지 않았으며 제한 시간은 5초로 표시돼 있다. 이는 **입력 화면의 상태**이며 저장 성공이나 질의 경로를 입증하지 않는다. 5초를 모든 환경의 권장값으로 일반화하지 않는다.

| 설정 | 결정할 질문 | 검증 |
|---|---|---|
| DNS Domain | 어느 접미사를 넘길 것인가? | 대상 이름·하위 이름 및 기존 로컬 영역과의 충돌 |
| MasterServers | 각 서버가 같은 이름 공간을 올바르게 답하는가? | 전달하는 DNS 서버에서 각 대상으로 직접 질의 |
| AD 저장·범위 | 어느 DNS에 같은 경로가 필요한가? | 대상 DNS별 전달 영역과 복제 상태 |
| 재귀 대체 경로 | 지정 대상 실패 시 다른 DNS로 보내도 되는가? | `UseRecursion`과 서버 전체 재귀·Root Hints 설정 |
| 시간 제한 | 후속 서버를 시도할 시간 여유가 있는가? | 영역·서버 제한 시간과 실제 캡처 시각 |

DNS Manager의 서버 이름 검증이 실패했다고 바로 대상 주소를 삭제하지 않는다. 이름 표시의 검증 문제인지, 실제 DNS 질의에 응답하지 않는지부터 구분한다. 원본의 “검증 실패 대상 제거” 절차는 원인 확인 이후의 변경 판단으로 제한한다. [설정 준비와 검증 절차](https://learn.microsoft.com/en-us/windows-server/networking/dns/configure-dns-forwarding#prepare-the-dns-server-for-dns-forwarding)

## 타임아웃과 NXDOMAIN을 나눈다

| 결과·조건 | 해석과 다음 확인 |
|---|---|
| NXDOMAIN | 해당 경로에서 이름이 없다는 최종 응답. 이름·권한 데이터 확인 |
| NODATA | 이름은 있으나 요청 유형의 데이터가 없을 수 있음. 유형과 응답 내용 확인 |
| 응답 없음 | 통신·대상 상태와 다음 서버 시도·제한 시간 확인 |
| REFUSED | 요청 거부. 재귀 허용·질의 출발 주소·정책 확인 |
| SERVFAIL | 해결 실패. 상위 응답·로그·DNSSEC·제한 시간 등으로 범위 축소 |

서버 수준 `ForwardingTimeout`, 조건부 전달 영역의 `ForwarderTimeout`, 서버 전체 `RecursionTimeout`은 같은 설정이 아니다. 대상 서버를 여러 개 넣어도 전체 처리 시간이 먼저 소진되면 뒤 서버까지 시도하지 못한다. Root Hints 대체 역시 재귀·허용 설정·네트워크·남은 시간의 조건을 받는다. [시간 제한과 실패 동작](https://learn.microsoft.com/en-us/windows-server/networking/dns/forwarding#dns-server-timeouts-and-failure-behavior)

사설 이름이 지정한 서버 밖으로 전달되면 정보가 노출되거나 다른 답을 받을 수 있다. 재귀 대체 여부를 명시하고 네트워크 경계에서도 질의 경로를 통제한다. 설정만으로 네트워크 보안 경계가 생기는 것은 아니다.

## 조회 위치를 맞춰 검증한다

설정 조회는 DnsServer 모듈이 있는 관리 PC에서 대상 서버 조회 권한으로 실행한다. 아래 값은 가상 예제이며 실제 실행 결과는 없다.

```powershell
Get-DnsServerForwarder -ComputerName DC01 | Format-List *
Get-DnsServerRecursion -ComputerName DC01 | Format-List *
Get-DnsServerZone -ComputerName DC01 -Name 'partner.example.net' |
    Format-List *
```

영역 유형과 `MasterServers`, `IsDsIntegrated`, `ForwarderTimeout`, `UseRecursion` 등을 기록한다. 화면에 없는 값을 기본값이라고 추정하지 않는다. [전달 설정 조회 항목](https://learn.microsoft.com/en-us/windows-server/networking/dns/configure-dns-forwarding#validate-dns-forwarding)

대상 DNS 직접 질의는 **실제로 전달하는 DNS 서버**의 PowerShell에서 수행한다. 관리 PC에서 성공한 질의는 DNS 서버의 출발 주소와 방화벽 경로를 검증하지 못한다. `192.0.2.53`은 문서용 주소이므로 준비한 테스트 대상 주소로 바꾼다. DnsClient 모듈과 해당 질의를 보낼 네트워크 권한이 필요하다.

```powershell
Resolve-DnsName -Name 'service.partner.example.net' -Type A -Server '192.0.2.53' -DnsOnly -NoHostsFile
Resolve-DnsName -Name 'service.partner.example.net' -Type A -Server '192.0.2.53' -DnsOnly -NoHostsFile -TcpOnly
```

일반 질의와 TCP 강제 질의를 비교한다. 일반 질의는 상황에 따라 TCP로 전환될 수 있으므로 명령 성공만으로 UDP 경로를 증명하지 않는다. UDP/TCP별 실제 전송은 캡처로 확인한다. [DNS 전달에 필요한 UDP·TCP 53과 직접 테스트](https://learn.microsoft.com/en-us/windows-server/networking/dns/configure-dns-forwarding#prepare-the-dns-server-for-dns-forwarding)

대표 클라이언트에서는 내부 DNS를 명시해 같은 이름을 확인한다.

```powershell
Resolve-DnsName -Name 'service.partner.example.net' -Type A -Server DC01 -DnsOnly -NoHostsFile
```

| 비교 | 좁혀지는 범위 |
|---|---|
| 대상 직접 질의부터 실패 | 출발 주소·네트워크·대상 서비스·데이터 |
| 직접 성공, 내부 DNS 경유 실패 | 로컬 영역·캐시·전달 대상·재귀·시간 제한 |
| DNS 서버에서 성공, 클라이언트에서 실패 | 클라이언트 설정·클라이언트↔DNS 경로 |
| 조회 성공, 전달 경로는 미확인 | 권한 영역·캐시·대체 경로 응답 가능. 경로 확인은 별도 |

특정 전달자를 거쳤다는 증거가 필요하면 승인된 테스트 이름과 캐시 조건을 준비하고 DNS 서버에서 패킷을 수집한다. 운영 캐시 전체 삭제는 다른 질의와 상위 트래픽에도 영향을 줄 수 있다. [응답 성공과 경로 검증의 구분](https://learn.microsoft.com/en-us/windows-server/networking/dns/configure-dns-forwarding#validate-dns-forwarding)

## 변경·원복은 경로 단위로 기록한다

변경 전 서버 목록, 대상 접미사, AD 저장·복제 범위, 재귀·Root Hints·시간 제한, 기존 조회 결과와 출발 위치를 남긴다. 한 번에 한 설정을 바꾸고 해당 이름 공간과 기존 내부 이름을 모두 검사한다. 양쪽 DNS가 서로에게 같은 이름을 되돌리는 루프도 점검한다.

대상 누락, 승인 밖 경로 사용, 기존 이름 실패, 지연 증가가 확인되면 확대를 중단한다. 주소만 되돌리지 말고 변경한 대체 경로·범위·시간 제한도 기록한 값으로 복원한 뒤 같은 질의를 재검증한다. 새 조건부 전달자를 제거할 때도 기존 영역과 의존 시스템을 먼저 확인한다. [변경 전 기록을 이용한 원복](https://learn.microsoft.com/en-us/windows-server/networking/dns/configure-dns-forwarding#roll-back-dns-forwarding)

관련 글: [AD 통합 영역의 저장·복제·갱신 검증]({% post_url 2026-08-31-ad-dns-integrated-zones %}).

---

적용 범위: Windows Server 2019 이상 DNS. 버전·재귀·정책 조건은 공식 문서 참조. 원본 화면·본문 및 공식 근거 대조: 2026-11-10. 실제 AD/DNS에서 명령 실행·패킷 수집·장애 재현은 하지 않았다.

[연재 목차]({% post_url 2026-08-13-ad-series-guide %})
