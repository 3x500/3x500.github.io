---
layout: single
title: "AD 05. AD 통합 DNS 영역과 동적 업데이트"
categories: [Active Directory]
tags: [AD, DNS, Dynamic Update]
toc: true
published: false
---

DNS 레코드가 어디에 저장되는지에 따라 복제와 변경 방법도 달라진다. AD 통합 영역은 영역 데이터를 AD DS에 저장하고 AD 복제로 전달하는 방식이다.

일반적인 파일 기반 주 영역이 원본 파일을 중심으로 관리된다면, AD 통합 영역은 해당 데이터를 보유한 쓰기 가능한 DNS/DC가 변경을 처리할 수 있다. 모든 DNS 서버에 자동으로 같은 영역이 생긴다는 뜻은 아니다. 복제 범위와 서버 역할이 맞아야 한다. [Microsoft DNS 영역](https://learn.microsoft.com/en-us/windows-server/networking/dns/zone-types)

## 영역의 세 설정을 함께 본다

| 설정 | 확인할 질문 |
|---|---|
| 저장 방식 | 파일 기반인가, AD 통합인가? |
| 복제 범위 | 어느 도메인·Forest·지정 파티션에 배포되는가? |
| 동적 업데이트 | 누가 레코드를 등록·갱신할 수 있는가? |

AD 통합 DNS는 DNS용 애플리케이션 디렉터리 파티션을 사용할 수 있다. 그래서 DNS 서비스 장애인지, AD 복제 문제인지도 구분해야 한다. 한 서버에서 레코드를 수정했는데 다른 서버에서 오래된 값이 보이면 복제 범위·복제 상태와 캐시를 나누어 확인한다. [Microsoft AD 통합 영역](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/active-directory-integrated-dns-zones)

## 보안 동적 업데이트의 의미

동적 업데이트는 주소가 바뀌었을 때 레코드를 자동으로 등록·갱신하는 기능이다. 보안 동적 업데이트는 AD 통합 영역에서 인증과 레코드 권한을 사용해 갱신을 통제한다. 레코드 소유권과 DHCP의 등록 방식에 따라 갱신 결과가 달라질 수 있다. [Microsoft 동적 업데이트](https://learn.microsoft.com/en-us/windows-server/networking/dns/dynamic-update)

갱신 오류를 해결하려고 `NonsecureAndSecure`로 바꾸면 등록 주체의 제한을 약화시킬 수 있다. 먼저 누가 해당 레코드를 소유하고 어떤 계정으로 갱신하는지 확인한다.

## 설정 조회 예제

DNS 관리 도구와 DnsServer 모듈이 있는 관리 PC에서, 대상 서버 조회 권한으로 실행한다. 실제 출력 대신 확인할 속성만 제시한다.

```powershell
Get-DnsServerZone -ComputerName DC01 |
    Select-Object ZoneName, ZoneType, IsDsIntegrated, DynamicUpdate
```

이 결과만으로 복제 범위까지 확인했다고 보지는 않는다. 영역 속성과 AD 복제 상태를 함께 확인한다. [Get-DnsServerZone](https://learn.microsoft.com/en-us/powershell/module/dnsserver/get-dnsserverzone)

레코드 목록을 파일로 내보내는 작업도 전체 AD 복구 전략과 구분한다. 영역 설정·보안 정보·디렉터리 데이터 복구까지 필요한지에 따라 백업 방법이 달라진다.

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.
