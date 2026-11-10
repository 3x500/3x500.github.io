---
layout: single
title: "AD 04. AD는 DNS로 DC를 어떻게 찾는가"
categories: [Active Directory]
tags: [AD, DNS, SRV, DC Locator]
toc: true
published: false
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

## 두 종류의 조회를 비교한다

도메인 클라이언트의 PowerShell에서 실행할 조회 예제다. 도메인과 서버 이름은 가상 값이며 실제 출력은 싣지 않았다.

```powershell
Resolve-DnsName -Name _ldap._tcp.dc._msdcs.ad.example.com -Type SRV
Resolve-DnsName -Name DC01.ad.example.com -Type A
```

첫 결과에서는 대상 서버와 포트를 보고, 두 번째에서는 해당 서버의 주소를 확인한다. 후보 이름은 반환되는데 주소가 오래된 값이라면 다음 연결 단계가 실패할 수 있다. [Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)

## 장애를 좁히는 순서

1. 클라이언트에 설정된 DNS 서버가 내부 도메인 레코드를 조회할 수 있는지 확인한다.
2. 필요한 SRV 레코드와 대상 서버의 주소 레코드를 확인한다.
3. 조회된 DC까지 필요한 통신이 가능한지 확인한다.
4. 특정 지점에서만 문제가 나면 Site·Subnet과 사이트별 검색도 확인한다.

공용 DNS를 보조 서버로 추가하는 것만으로 AD용 DNS 이중화가 되지는 않는다. 내부 AD 영역을 모르는 서버에 질의하면 서비스 검색이 실패할 수 있다. 이중화 대상은 같은 내부 이름을 올바르게 해결할 수 있어야 한다.

레코드를 수동으로 추가하기 전에 DC의 등록 상태와 DNS 영역·복제 상태를 점검한다. 레코드 한 개를 보충해 증상이 사라져도 원래 등록 문제가 남아 있을 수 있다.

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.
