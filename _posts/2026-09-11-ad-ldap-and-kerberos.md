---
layout: single
title: "AD 08. LDAP 연동 진단: 연결·TLS·Bind·Search·ACL 분리하기"
categories: [Active Directory]
tags: [AD, LDAP, Kerberos, LDAPS]
toc: false
toc_sticky: true
published: true
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

## 원본 LDS 연동 절차를 다섯 단계로 검증한다

원본 LDS 문서의 서버·포트·인증 방식·Base DN 설정은 AD DS 앱 연동을 설명할 때도 유용하다. 다만 연결한 인스턴스가 다르면 파티션·스키마·계정과 권한도 다를 수 있다. AD 관리 모듈의 성공을 앱의 LDAP 인증 성공으로 치환하지 않는다.

| 단계 | 성공 근거 | 실패 시 확인 |
|---|---|---|
| 연결 | 의도한 FQDN·포트의 서비스 응답 | 이름·네트워크·리스너·실제 인스턴스 |
| TLS | 서버 이름·체인·유효기간을 검증한 보안 세션 | 인증서·신뢰 저장소·클라이언트 지원·중간 종료 지점 |
| Bind | 선택한 인증 방식과 신원으로 성공 | 계정 형식·상태·자격 증명·서명/CBT 정책 |
| Search | 정확한 Base DN·Scope·Filter의 기대 객체 | 파티션·객체 위치·필터 구문·읽기 속성 |
| ACL | 해당 계정으로 필요한 읽기/쓰기가 허용 | 권한·상속·보호 객체·거부 항목 |

원본 D-049의 389 연결 화면과 D-050/051의 Simple Bind 예제는 TLS 보호가 확인되지 않는다. 이 글에서는 LDAPS 성공 증거로 사용하지 않는다. Simple Bind는 반드시 TLS 등 적절한 전송 보호를 확인해야 하며, 포트가 열렸다는 사실은 인증서 검증을 대신하지 않는다. [LDAP 보호 계층](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing#what-ldap-signing-does-and-doesnt-do)

## 같은 조건으로 허용·거부를 확인한다

테스트 도구나 앱에서 다음 연결표를 작성한다. 도구 사용 계정과 앱 서비스 계정이 다르면 같은 권한 검증으로 취급하지 않는다. 운영 암호·토큰은 연결표·스크린샷·로그에 남기지 않는다.

| 항목 | 설명용 값·결정 사항 |
|---|---|
| 서비스 | AD DS인지 AD LDS 특정 인스턴스인지 명시 |
| Endpoint | `DC01.ad.example.com`, 준비한 TLS 포트 |
| 인증 | 앱이 지원하는 SASL/Kerberos 또는 TLS 보호 Simple Bind |
| Base DN | `OU=Users,DC=ad,DC=example,DC=com` |
| Scope·Filter | 필요한 하위 범위와 객체 조건, 특수문자 이스케이프 |
| 속성·작업 | 필요한 속성 읽기, 승인된 경우만 쓰기 |

정상 계정·오류 계정, 범위 안·밖 객체, 허용 읽기·비허용 쓰기를 나눠 관찰한다. Bind 성공 후 빈 검색 결과는 검색 조건·읽기 권한 문제일 수 있다. 반대로 관리 도구로 조회된 계정이 앱에 로그인하지 못한다면 앱의 신원 매핑·권한 부여 로직도 확인한다. DN 이스케이프와 필터 이스케이프 규칙을 혼용하거나 사용자 입력을 LDAP 필터에 그대로 이어붙이지 않는다.

## 변경은 호환성과 권한 범위를 함께 관리한다

TLS·서명·CBT를 강제하기 전 클라이언트와 인증 방식을 목록화한다. 서비스 계정, 포트, 검색 범위, 정책을 한꺼번에 바꾸지 않는다. 변경 후 인증·검색·기능별 권한과 비승인 거부를 검증하고, 장애가 생기면 해당 단계의 이전 설정으로 복원한 뒤 같은 조건으로 재검사한다. 인증서 오류를 무시하는 코드나 관리자 권한 부여를 호환성 해결책으로 넣지 않는다.

## 연동 계정에는 필요한 권한만 준다

사용자 조회만 하는 애플리케이션에 사용자 수정이나 관리자 권한까지 줄 필요는 없다. 검색 범위와 읽을 속성을 정의하고, 조회 권한과 계정 관리 권한을 구분한다. 연결 문자열에 운영 암호를 적어 공유하는 방식도 피한다.

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.

[연재 목차]({% post_url 2026-08-13-ad-series-guide %})
