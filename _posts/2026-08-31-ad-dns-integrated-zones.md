---
layout: single
title: "AD 05. AD 통합 DNS 설계: 복제 범위와 갱신 권한을 함께 검증하기"
categories: [Active Directory]
tags: [AD, DNS, Dynamic Update, Security]
toc: true
toc_sticky: true
published: false
---

DNS 서버 두 대가 같은 레코드를 응답해도 저장 방식과 갱신 권한까지 올바르다고 판단할 수는 없다. AD 통합 영역을 설계할 때는 **어디에 저장하고, 어느 서버로 복제하며, 누가 갱신하는지**를 함께 정해야 한다.

이 글은 구축·DNS 운영 문서의 영역 생성 화면을 바탕으로 설계와 검증 순서를 재구성했다. 화면의 식별정보는 검정색으로 가렸으며 설정과 출력값은 보존했다. 도메인 `ad.example.com`, 서버 `DC01`·`DC02`, 호스트 `app01`은 본문 설명용 가상 이름으로, 화면의 실제 이름을 대체한 실행 결과가 아니다.

## 요구를 저장·복제·갱신으로 나눈다

설명용 요구는 내부 이름을 두 DNS/DC에서 제공하고, 승인된 장비 또는 DHCP 갱신 계정이 주소 변경을 등록하는 것이다. 두 서버는 같은 영역을 보유한 쓰기 가능한 DC이며 DNS Server 역할이 설치돼 있다고 가정한다. 서버 두 대를 둬도 같은 가상화 호스트나 스토리지에 의존하면 장애 범위가 겹친다. 서버 수와 장애 도메인은 별도로 설계한다.

![클라이언트 질의와 보안 갱신, DNS/DC 사이의 AD 복제를 구분한 설명용 구성도]({{ '/images/ad-deep-dive/dns-integrated-flow.svg' | relative_url }})

그림의 실선은 질의·응답, 녹색 경로는 보안 갱신, 파란색 경로는 영역 데이터의 AD 복제다. DNS 응답이 돌아오는 경로와 디렉터리 데이터가 수렴하는 경로를 나눠 관찰한다.

| 선택 | 쓰기 위치와 전달 방식 | 선택할 조건·비용 |
|---|---|---|
| 파일 기반 Primary | 주 서버에서 변경, 다른 서버에는 영역 전송 | AD 통합을 사용할 수 없는 환경. 주 서버의 변경 가용성과 전송 대상을 관리 |
| Secondary | 다른 서버에서 받은 읽기 전용 사본 | 다른 DNS 구현과 읽기 사본 공유. 자체 갱신 원본으로 사용하지 않음 |
| AD 통합 Primary | 영역을 보유한 쓰기 가능한 DNS/DC에서 변경, AD 복제 | AD의 복제와 보안 갱신을 활용. 디렉터리 복제 상태까지 운영 책임에 포함 |

AD 통합 영역은 DNS 역할이 설치된 DC에서 제공한다. 일반 멤버 서버에 DNS 역할만 추가해서 AD 통합 저장소를 갖게 되지는 않는다. Secondary의 영역 전송과 AD 통합 영역의 AD 복제도 구분한다. [Microsoft DNS 영역의 저장 방식과 유형](https://learn.microsoft.com/en-us/windows-server/networking/dns/zone-types)

## GUI 선택을 운영 설정과 연결한다

원본의 생성 절차에서 중요한 화면 세 장을 골랐다. 화면에서 옵션을 선택했다는 사실과 생성 후 해당 설정이 적용됐다는 사실은 별도다.

![주 영역과 Active Directory 저장 옵션을 선택한 원본 설정 화면]({{ '/images/ad-deep-dive/b-037.png' | relative_url }})

Primary zone과 Store the zone in Active Directory가 함께 선택돼 있다. Primary라는 이름만 보고 파일 기반이라고 판단하지 않는다.

![도메인의 DNS DC로 복제하는 옵션을 선택한 화면. 도메인명 세 곳은 가림]({{ '/images/ad-deep-dive/b-038.png' | relative_url }})

도메인 내 DNS/DC를 대상으로 하는 옵션이 선택돼 있다. 이 화면만으로 대상 서버가 실제 파티션에 참여하고 복제를 완료했는지는 알 수 없다.

| GUI·속성 | 설계에서 정할 것 | 생성 후 확인 |
|---|---|---|
| Zone Type / AD 저장 | AD 통합 여부 | `IsDsIntegrated`와 `ZoneType` |
| Replication Scope | Domain·Forest·사용자 지정 범위 | 영역 속성의 범위와 디렉터리 파티션, 대상 DC의 참여 |
| Dynamic Update | Secure 또는 갱신 금지 등 요구에 맞는 정책 | `DynamicUpdate`와 실제 승인 갱신 결과 |
| 레코드 Security | 갱신 주체·소유권·ACL | 장비/DHCP 등록 계정과 기존 레코드 권한 |

보통 도메인 DNS용 `DomainDnsZones`, Forest DNS용 `ForestDnsZones` 같은 애플리케이션 파티션을 사용한다. 모든 DC 또는 Forest의 모든 DNS가 무조건 모든 영역을 받는다는 뜻은 아니다. 범위를 넓히면 필요한 DNS 서비스에 데이터를 배포할 수 있지만 복제 대상과 운영 영향도 늘어난다. 사용자 지정 파티션은 참여 대상까지 관리해야 한다. [DNS 애플리케이션 파티션 조회](https://learn.microsoft.com/en-us/powershell/module/dnsserver/get-dnsserverdirectorypartition?view=windowsserver2025-ps)

## 보안 동적 업데이트는 갱신 주체까지 확인한다

![보안 동적 업데이트만 허용하도록 선택한 원본 설정 화면]({{ '/images/ad-deep-dive/b-041.png' | relative_url }})

Allow only secure dynamic updates가 선택돼 있다. AD 통합 영역의 보안 갱신은 인증된 보안 컨텍스트와 레코드 권한을 사용한다. 인증 성공만으로 모든 기존 레코드를 덮어쓸 수 있는 것은 아니다. 영역·레코드 ACL은 관리와 갱신 접근을 통제하며 일반 DNS 질의를 제한하는 설정과는 구분한다. [보안 동적 업데이트와 ACL](https://learn.microsoft.com/en-us/windows-server/networking/dns/dynamic-update#secure-dynamic-update)

| 등록 경로 | 확인할 질문 | 흔한 실패 분기 |
|---|---|---|
| 장비가 직접 등록 | 어떤 컴퓨터 계정이 기존 레코드를 소유하는가? | 장비 재설치·이름 재사용 후 이전 소유권과 충돌 |
| DHCP가 대신 등록 | A/PTR 각각 누가 갱신하며 어떤 자격 증명을 쓰는가? | 다른 DHCP 서버나 계정으로 바뀌어 기존 레코드 수정 거부 |
| 관리자가 수동 생성 | 이후 자동 갱신을 허용할 대상인가? | 수동 레코드의 ACL과 자동 등록 주체가 불일치 |

갱신 거부를 해결하려고 곧바로 비보안 갱신을 허용하지 않는다. DNS Client와 DHCP의 등록 분담, 기존 소유자와 ACL, 갱신 요청의 오류부터 확인한다. DHCP용 갱신 계정을 쓰는 설계에서는 해당 계정의 수명주기와 자격 증명 변경도 운영 대상이다.

DHCP가 등록을 맡는 장비에 `ipconfig /registerdns`를 일괄 실행하면 클라이언트가 직접 등록을 시도해 소유권 운영에 영향을 줄 수 있다. 임대 갱신 역시 통신에 영향을 줄 수 있으므로 테스트 장비에서 승인된 절차로 확인한다. [DNS와 DHCP의 갱신 동작 및 주의사항](https://learn.microsoft.com/en-us/windows-server/networking/dns/dynamic-update#how-computers-update-their-dns-names)

Secondary 등 일반 영역 전송이 필요한 경우 허용할 **DNS 서버·주소**를 제한한다. 이를 “허용 도메인 목록”이나 일반 질의 ACL로 설명하지 않는다. AD 통합 영역의 DC 간 AD 복제와 별도 DNS 서버에 제공하는 영역 전송도 분리해 검증한다. [영역 전송 대상 제한](https://learn.microsoft.com/en-us/windows-server/networking/dns/zone-types#zone-transfer-settings)

## 서버를 지정해 설정과 데이터를 따로 조회한다

아래는 **실행 예제**다. 이 글 작성 중 실제 AD에서 실행하지 않았다. DnsServer 관리 모듈을 설치한 관리 PC에서 대상 서버를 조회할 수 있는 위임 권한으로 실행한다. `DC01`·`DC02`는 실제 대상 이름으로 바꾼다. 이름 확인부터 문제가 있으면 검증된 서버 주소를 사용한다. 조회 오류나 빈 값은 정상으로 처리하지 않는다.

```powershell
Get-DnsServerZone -ComputerName DC01 -Name 'ad.example.com' |
    Format-List *
Get-DnsServerZone -ComputerName DC02 -Name 'ad.example.com' |
    Format-List *

Get-DnsServerDirectoryPartition -ComputerName DC01
Get-DnsServerDirectoryPartition -ComputerName DC02
```

영역 출력에서 `ZoneType`, `IsDsIntegrated`, `DynamicUpdate`를 비교한다. 환경에서 노출되는 파티션 관련 속성과 DNS Manager의 영역 속성도 함께 확인한다. 파티션 목록에 이름이 보이는 것만으로 특정 영역의 복제 범위가 맞다고 결론내리지 않는다. [Get-DnsServerZone의 대상 서버·영역 매개변수](https://learn.microsoft.com/en-us/powershell/module/dnsserver/get-dnsserverzone?view=windowsserver2025-ps)

![원본 조회 결과 중 Primary, IsDsIntegrated True가 표시된 화면. 이름과 계정 경로는 가림]({{ '/images/ad-deep-dive/b-049.png' | relative_url }})

원본 출력은 `Primary`, `IsDsIntegrated=True`, `IsReverseLookupZone=False`, `IsSigned=False`를 보여준다. **AD 통합 정방향 영역**이라는 근거로 사용할 수 있지만, 이 화면에는 DynamicUpdate나 복제 범위가 없고 두 서버 간 수렴도 증명하지 않는다. `IsSigned=False`는 DNSSEC 서명 상태이며 보안 동적 업데이트 허용 여부와 다른 속성이다.

영역 데이터와 질의 결과는 다음처럼 구분해 본다. 앞 두 명령은 DnsServer 모듈·서버 조회 권한이 필요하다. 뒤 두 명령은 DnsClient 모듈이 있는 Windows PC에서 일반 DNS 질의로 실행한다.

```powershell
Get-DnsServerResourceRecord -ComputerName DC01 -ZoneName 'ad.example.com' -Name app01
Get-DnsServerResourceRecord -ComputerName DC02 -ZoneName 'ad.example.com' -Name app01

Resolve-DnsName -Name 'app01.ad.example.com' -Type A -Server DC01 -DnsOnly -NoHostsFile
Resolve-DnsName -Name 'app01.ad.example.com' -Type A -Server DC02 -DnsOnly -NoHostsFile
```

서버 조회는 저장된 레코드, DNS 질의는 지정한 서버의 응답을 관찰한다. 응답 순서나 캐시로 줄어든 TTL 차이만으로 복제 장애라고 판단하지 않는다. CNAME·추가 레코드가 섞이면 같은 이름·유형의 주소 집합을 비교한다. [Resolve-DnsName의 서버 지정과 DNS 전용 질의](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname?view=windowsserver2025-ps)

## 결과에 따라 다음 확인을 고른다

| 관찰 | 다음 확인 | 이 단계에서 단정하지 않을 것 |
|---|---|---|
| 한 DNS에 영역 자체가 없음 | 저장 방식, 복제 범위, 대상 서버 역할·파티션 참여, AD 복제 | 캐시 삭제로 해결된다고 단정 |
| 두 서버의 저장 레코드가 다름 | 변경 시각, 해당 파티션의 복제 파트너·마지막 성공·오류 | 질의 한 번으로 전체 AD 복제 장애 확정 |
| 저장 레코드는 같은데 장비만 이전 주소 사용 | 장비 DNS 설정, Hosts·캐시, 실제 질의 서버와 응답 | 영역 삭제·재생성 |
| 조회는 되지만 갱신이 거부됨 | 갱신 주체, 인증·시간·레코드 소유권과 ACL | 비보안 갱신으로 전환 |
| 타임아웃 또는 SERVFAIL | 대상 도달성, DNS 서비스·로그, 영역 로드 상태 | 응답 없음과 이름 없음(NXDOMAIN)을 같은 원인으로 처리 |

AD 복제 상세 조사가 필요하면 관리 도구와 권한을 갖춘 관리 호스트에서 `repadmin /showrepl DC02`로 파티션별 상태를 읽는다. 원본에 있는 DNS 이벤트나 `dcdiag` 결과 한 줄은 원인 후보를 좁히는 단서다. 특정 이벤트 번호 또는 단일 PASS만으로 전체 정상·고장을 확정하지 않는다. [Repadmin /showrepl](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/cc742066(v=ws.11))

## 변경과 원복 기준을 먼저 정한다

변경 전에는 서버별 영역 속성, 복제 범위, 관련 레코드·소유권·ACL, DHCP 등록 주체와 검증 시각을 기록한다. 레코드 CSV는 비교 자료로 쓸 수 있지만 영역 보안·디렉터리 상태를 복원하는 전체 백업은 아니다. [DNS 영역 내보내기의 범위](https://learn.microsoft.com/en-us/powershell/module/dnsserver/export-dnsserverzone?view=windowsserver2025-ps)

| 변경 | 적용 후 확인 | 중단·원복 판단 |
|---|---|---|
| 복제 범위 조정 | 필요한 DNS가 영역을 로드하고 해당 파티션 복제가 수렴 | 대상 누락·예상 밖 범위 확장 시 추가 변경 중단, 기록한 이전 범위 복원 계획 적용 |
| Secure 정책 적용 | 승인 장비·DHCP 갱신 성공과 비승인 갱신 거부를 테스트 | 정상 갱신 영향 시 대상·ACL·인증을 조사하고 파일럿 확대 보류 |
| 레코드 권한 조정 | 승인 주체가 필요한 레코드만 갱신 | 권한 확장·갱신 중단 시 기록한 ACL을 기준으로 복구 |

공유되는 AD 통합 영역의 삭제는 복제로 다른 서버에도 영향을 줄 수 있다. 단순한 원복 방법으로 영역 삭제·재생성을 사용하지 않는다. 격리된 테스트 영역에서 요구·설정·허용/거부 결과를 먼저 맞추고 운영 변경을 진행한다.

---

적용 범위: Windows Server 2019 이상 DNS/AD DS의 설계·조회. 기능·도구 조건은 각 공식 문서 참조. 원본 화면 검토 및 공식 본문 대조: 2026-11-10. 현재 환경의 AD 명령 실행·갱신·장애 재현 결과는 포함하지 않았다.

[연재 목차]({% post_url 2026-08-13-ad-series-guide %})
