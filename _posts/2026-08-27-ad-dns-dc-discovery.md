---
layout: single
title: "AD 04. DC Locator 진단: 레코드·후보 응답·Site·실제 통신 구분하기"
categories: [Active Directory]
tags: [AD, DNS, SRV, DC Locator]
toc: false
toc_sticky: true
published: true
---

웹사이트가 열린다고 AD에서 필요한 DNS도 정상이라고 볼 수는 없다. 인터넷 이름을 조회하는 것과 도메인의 인증 서비스를 찾는 것은 다른 질의다.

DNS를 안내소로 비유하면 A 레코드는 서버의 주소를, SRV 레코드는 특정 업무를 맡는 서버와 포트를 알려준다. 도메인에 가입하거나 DC를 찾는 과정에서는 AD용 SRV 레코드가 중요하다. [Microsoft SRV 검증](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/verify-srv-dns-records-have-been-created)

## DC 검색의 큰 흐름

```text
클라이언트 → DNS에서 서비스 후보 검색
           → 후보 DC의 응답·기능 확인
           → Site 정보 등을 반영해 DC 선택
```

SRV 응답이 곧 모든 통신의 성공을 뜻하지는 않는다. 조회된 DC로 실제 연결할 수 있어야 하고, 요청한 서비스를 제공해야 한다. DC Locator는 DNS 조회에 더해 DC 응답과 Site 정보 등을 사용한다. [Microsoft DC Locator](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/dc-locator)

## 원본 레코드 표를 서비스 검색에 연결한다

원본 DNS 운영 문서의 레코드 목록은 호스트 접근, 역방향 확인, 별칭, 서비스 위치를 구분한다. 원본의 애플리케이션 SRV 수동 생성 화면은 DC의 자동 등록 성공 화면으로 사용하지 않는다.

| 유형 | 제공하는 정보 | 증명하지 못하는 것 |
|---|---|---|
| A·AAAA | 이름의 IPv4·IPv6 주소 | 해당 주소의 인증·LDAP 서비스 정상 |
| PTR | 주소의 역방향 이름 | 호출자의 신뢰나 Kerberos 인증 |
| CNAME | 다른 정식 이름을 가리키는 별칭 | 별칭에 맞는 SPN·인증서·앱 설정 |
| SRV | 서비스 후보 이름·포트·우선순위·가중치 | 실제 후보의 가용성·권한·최종 인증 성공 |

후보 DC는 Netlogon의 DNS 등록과 서비스 기능에 연결된다. 클라이언트는 DNS 기반 검색 후 UDP LDAP 응답 등을 확인하고 결과를 캐시한다. Site 정보가 최적 후보 검색에 쓰이지만 현재 같은 Site DC를 선택하지 않았다는 이유만으로 장애라고 단정하지 않는다. [DC Locator의 검색과 Site 선택](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/dc-locator)

## 두 종류의 조회를 비교한다

도메인 클라이언트의 PowerShell에서 실행할 조회 예제다. 도메인과 서버 이름은 가상 값이며 실제 출력은 싣지 않았다.

```powershell
Resolve-DnsName -Name _ldap._tcp.dc._msdcs.ad.example.com -Type SRV -Server DC01 -DnsOnly -NoHostsFile
Resolve-DnsName -Name DC01.ad.example.com -Type A -Server DC01 -DnsOnly -NoHostsFile
```

첫 결과에서는 대상 서버와 포트를 보고, 두 번째에서는 해당 서버의 주소를 확인한다. 후보 이름은 반환되는데 주소가 오래된 값이라면 다음 연결 단계가 실패할 수 있다. [Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)

## 후보 응답과 TCP 연결을 별도로 확인한다

DNS 질의에는 Windows DnsClient 모듈이 필요하며 일반 사용자로 실행할 수 있다. 대상 DNS 주소와 질의 이름은 환경에 맞춰 바꾼다. IPv6를 쓰면 AAAA와 해당 통신도 확인한다. DC Locator 진단은 대상 도메인에 대한 통신이 가능한 Windows 클라이언트에서 실행한다.

```powershell
nltest /dsgetdc:ad.example.com
nltest /dsgetsite
Test-NetConnection -ComputerName DC01.ad.example.com -Port 389
```

`nltest`의 DC·Site·기능 정보를 요청한 도메인과 비교한다. 캐시된 선택과 새 검색은 구분하고 필요 시 승인된 진단에서 재검색 조건을 사용한다. `Test-NetConnection`의 성공은 해당 TCP 포트 연결의 관찰일 뿐 UDP LDAP ping, Bind, Kerberos, RPC 동적 포트, 사용자 로그인까지 검증하지 않는다. 원본의 포트 성공·실패 화면을 AD 전체 판정으로 확대하지 않는 이유다.

| 단계 | 성공 관찰 | 실패 시 다음 확인 |
|---|---|---|
| DNS 서버 선택 | 단말이 내부 영역을 해결할 DNS 사용 | NIC·VPN·DHCP 설정과 실제 질의 서버 |
| SRV 검색 | 필요한 서비스의 후보·포트 반환 | 영역·등록·복제·질의 이름 |
| 후보 주소 | 반환된 후보의 A/AAAA가 맞음 | 오래된 레코드·다중 NIC·주소 경로 |
| 후보·Site | Locator가 사용 가능한 DC와 Site 정보 확인 | UDP LDAP 경로·서비스·Subnet 매핑·캐시 |
| 실제 업무 | 인증·정책·리소스의 목적별 테스트 성공 | 시간·SPN·보안 채널·프로토콜·ACL |

## 장애를 좁히는 순서

1. 클라이언트에 설정된 DNS 서버가 내부 도메인 레코드를 조회할 수 있는지 확인한다.
2. 필요한 SRV 레코드와 대상 서버의 주소 레코드를 확인한다.
3. 조회된 DC까지 필요한 통신이 가능한지 확인한다.
4. 특정 지점에서만 문제가 나면 Site·Subnet과 사이트별 검색도 확인한다.

공용 DNS를 보조 서버로 추가하는 것만으로 AD용 DNS 이중화가 되지는 않는다. 내부 AD 영역을 모르는 서버에 질의하면 서비스 검색이 실패할 수 있다. 이중화 대상은 같은 내부 이름을 올바르게 해결할 수 있어야 한다.

레코드를 수동으로 추가하기 전에 DC의 등록 상태와 DNS 영역·복제 상태를 점검한다. Netlogon.dns의 등록 대상과 DNS Manager의 실제 레코드를 대조할 수 있다. 레코드 한 개를 보충해 증상이 사라져도 원래 등록 문제가 남아 있을 수 있다. [SRV 생성 검증의 세 방법](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/verify-srv-dns-records-have-been-created)

변경 전 단말 DNS 목록, 조회 결과, 선택 DC·Site와 해당 레코드를 기록한다. 파일럿 단말에서 DNS 경로를 조정한 뒤 기존 내부 이름과 도메인 작업을 함께 검증한다. 문제가 확대되면 기록한 설정으로 되돌리고 재검증한다. 영역·SRV 일괄 삭제나 보안 채널 초기화부터 시도하지 않는다.

버전 조건도 남긴다. Windows Server 2025의 DC Locator는 기본적으로 NetBIOS 기반 검색을 허용하지 않는다. 예전 단일 이름·WINS 동작을 최신 환경의 정상 경로로 가정하지 말고 DNS 이름과 공식 정책 조건을 확인한다. [NetBIOS 검색 정책](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/dc-locator#netbios-discovery-policy-setting)

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.

[연재 목차]({% post_url 2026-08-13-ad-series-guide %})
