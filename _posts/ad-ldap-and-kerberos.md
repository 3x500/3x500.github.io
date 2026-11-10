---
layout: single
title: "AD 08. LDAP와 Kerberos는 역할이 어떻게 다른가"
categories: [Active Directory]
tags: [AD, LDAP, Kerberos, LDAPS]
toc: true
published: false
---

LDAP와 Kerberos를 서로 대체하는 인증 제품처럼 비교하면 AD 연동이 헷갈린다. Kerberos는 티켓을 사용해 신원을 확인하는 인증 프로토콜이고, LDAP는 디렉터리에 연결해 정보를 검색·조작하는 프로토콜이다. LDAP 연결 과정에서도 인증이 필요할 수 있다.

문서함에 정보를 요청하는 방식이 LDAP라면, 요청자의 신원을 확인하는 수단 중 하나가 Kerberos인 셈이다. 두 역할은 한 연결 안에서 함께 사용될 수 있다. [Microsoft Kerberos 개요](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview), [Microsoft ADSI 인증](https://learn.microsoft.com/en-us/windows/win32/adsi/authentication)

## 애플리케이션 연결은 세 단계로 나눈다

```text
서버 연결 → 바인드(인증 방식 적용) → 지정 범위에서 검색
```

| 설정 | 확인할 내용 |
|---|---|
| 서버·포트 | 어느 디렉터리 인스턴스에 연결하는가? |
| 인증 방식 | 통합 인증인가, 자격 증명을 전달하는 방식인가? |
| Base DN·범위 | 어느 위치에서 어디까지 검색하는가? |
| 필터·속성 | 어떤 객체를 찾고 어떤 정보를 읽는가? |

연결 성공, 바인드 성공, 검색 성공은 서로 다른 확인이다. 검색 결과가 없으면 비밀번호보다 Base DN과 필터를 먼저 의심해야 할 수도 있다.

## LDAPS는 별도 디렉터리 제품이 아니다

LDAPS는 TLS로 보호하는 LDAP 연결이다. TLS는 전송 구간의 기밀성·무결성과 서버 인증을 제공한다. LDAP 서명과 채널 바인딩은 목적과 적용 조건이 다르다. `389` 포트만 보고 평문이라고 단정해서도 안 된다. STARTTLS나 SASL 보호 여부를 함께 확인해야 한다. [Microsoft LDAP 보호 계층](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing)

## 검색 시작 위치 확인 예제

Active Directory 모듈이 설치된 관리 PC에서 RootDSE 정보를 읽는 PowerShell 예제다. 실제 실행 결과는 포함하지 않았다.

```powershell
Get-ADRootDSE -Server DC01.ad.example.com |
    Select-Object DefaultNamingContext, ConfigurationNamingContext,
                  SchemaNamingContext
```

AD가 제공하는 명명 컨텍스트를 이해하는 데 쓸 수 있다. 단, 이 명령은 Active Directory 모듈을 통한 조회다. 애플리케이션의 LDAP·LDAPS 바인드나 인증서 검증이 성공했다는 증거로 사용할 수는 없다. 애플리케이션과 같은 서버·포트·인증 방식으로 별도 확인해야 한다. [Get-ADRootDSE](https://learn.microsoft.com/en-us/powershell/module/activedirectory/get-adrootdse)

## 연동 계정에는 필요한 권한만 준다

사용자 조회만 하는 애플리케이션에 사용자 수정이나 관리자 권한까지 줄 필요는 없다. 검색 범위와 읽을 속성을 정의하고, 조회 권한과 계정 관리 권한을 구분한다. 연결 문자열에 운영 암호를 적어 공유하는 방식도 피한다.

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.
