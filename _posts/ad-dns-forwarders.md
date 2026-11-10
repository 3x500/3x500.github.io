---
layout: single
title: "AD 06. Forwarder와 Conditional Forwarder"
categories: [Active Directory]
tags: [AD, DNS, Forwarder]
toc: true
published: false
---

내부 DNS가 모든 도메인의 레코드를 보유할 필요는 없다. 자신이 해결할 수 없는 질의를 다른 DNS 서버로 보내는 방법이 전달이다.

일반 전달자가 기본 문의 창구라면 조건부 전달자는 특정 분야의 전담 창구에 가깝다. 어느 이름에 대해 누구에게 질의하는지가 차이다. [Microsoft DNS 전달 설정](https://learn.microsoft.com/en-us/windows-server/networking/dns/configure-dns-forwarding)

## 두 전달자의 사용 예

| 종류 | 적용 예 |
|---|---|
| 서버 수준 Forwarder | 로컬 영역·캐시 등으로 해결하지 못한 일반 질의를 지정 DNS에 전달 |
| Conditional Forwarder | `partner.example.net` 질의를 그 이름 공간을 처리하는 DNS에 전달 |

조건부 전달자는 레코드를 복사해 보관하는 Secondary Zone과 다르다. 질의를 넘길 경로를 설정하는 것이므로 대상 서버가 응답하고 올바른 결과를 제공해야 한다.

## 실패하면 항상 다음 경로로 가는 것은 아니다

질의가 실패했다고 무조건 일반 전달자와 Root Hints를 차례로 사용하는 것으로 외우면 안 된다. 응답 종류, 재귀 설정, 전달자 시간 제한과 전체 재귀 시간 제한에 따라 동작이 달라진다.

예를 들어 대상 서버가 이름이 없다는 최종 응답을 반환하는 경우와 아예 응답하지 않는 경우는 다르게 봐야 한다. 전달자를 여러 개 등록했다는 사실만으로 모든 서버를 시도한다고 단정하지 않는다. [Microsoft 전달과 대체 경로](https://learn.microsoft.com/en-us/windows-server/networking/dns/forwarding)

## 조회 경로를 나눠 확인한다

다음은 관리 PC의 PowerShell에서 사용하는 가상 환경 조회 예제다.

```powershell
Get-DnsServerForwarder -ComputerName DC01
Get-DnsServerZone -ComputerName DC01 |
    Where-Object ZoneType -eq 'Forwarder'
Resolve-DnsName -Name service.partner.example.net -Server DC01.ad.example.com
```

서버 관리 조회에는 DnsServer 모듈과 대상 서버 조회 권한이 필요하다. 마지막 질의는 레코드가 실제로 준비된 예제 환경에서만 답을 얻을 수 있다. [Get-DnsServerForwarder](https://learn.microsoft.com/en-us/powershell/module/dnsserver/get-dnsserverforwarder)

사용자 PC에서 내부 DNS를 거친 결과와, 관리자가 대상 DNS에 직접 질의한 결과를 비교하면 전달 경로 문제를 좁히기 쉽다. 직접 조회는 성공하지만 전달 조회는 실패한다면 방화벽·재귀·전달 설정·캐시를 확인한다.

## 원복 기준도 남긴다

변경 전 대상 도메인, 전달 서버, 복제 여부와 기존 조회 결과를 기록한다. 변경 후에는 외부 이름 하나만 조회하지 말고 해당 조건부 도메인과 기존 내부 이름이 모두 정상인지 확인한다. 여러 DNS가 서로에게 같은 질의를 되돌리는 전달 루프도 피해야 한다.

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.
