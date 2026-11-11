---
layout: single
title: "AD 03. 객체 식별과 스키마: OU 이동·검색 범위·연동 영향 판단하기"
categories: [Active Directory]
tags: [AD, Schema, LDAP, DN]
toc: true
published: true
---

AD 관리 화면에서 사용자를 하나 만들면 디렉터리에 사용자 객체가 생긴다. 이름, 로그인 이름, 부서 같은 값은 그 객체의 속성이다. 스키마는 어떤 종류의 객체를 만들고 어떤 속성을 사용할 수 있는지 정하는 규칙이다.

표에 비유하면 객체는 한 행, 속성은 열에 가깝다. 다만 AD는 계층 구조와 객체 간 관계, 보안 정보를 가진 디렉터리이므로 일반적인 엑셀 표와 동일하지는 않다. [Microsoft 스키마 구현](https://learn.microsoft.com/en-us/windows/win32/ad/schema-implementation)

## DN은 객체가 놓인 경로다

다음은 설명용 사용자의 Distinguished Name(DN)이다.

```text
CN=user01,OU=Users,DC=ad,DC=example,DC=com
```

왼쪽부터 객체 이름과 상위 컨테이너를 따라 읽는다. 여기서 `DC=`는 Domain Component다. 서버 역할인 Domain Controller의 약어와 문맥이 다르다.

LDAP 주소에 서버와 DN을 함께 지정할 수도 있다. 애플리케이션의 검색 시작 위치인 Base DN과 사용자가 입력하는 로그인 이름은 별도 설정이다. 두 값을 혼동하면 계정이 존재하는데 검색되지 않는 문제가 생긴다. [Microsoft LDAP 경로](https://learn.microsoft.com/en-us/windows/win32/adsi/ldap-adspath)

## 이름과 식별자를 구분한다

객체를 다른 OU로 이동하면 DN이 바뀐다. 따라서 DN을 영구히 고정된 식별자로 가정한 외부 연동은 이동 작업의 영향을 받을 수 있다. 객체 식별에 어떤 값을 쓰는지는 연동 시스템의 사양과 함께 확인해야 한다.

사용자에게 보이는 표시 이름, 로그인에 쓰는 이름, 디렉터리의 DN도 서로 같은 값이라고 가정하지 않는다.

## 조회 예제

Active Directory PowerShell 모듈이 설치된 관리 PC에서, 해당 사용자를 읽을 수 있는 계정으로 실행하는 예제다. 실제 실행 결과는 포함하지 않았다.

```powershell
Get-ADUser -Identity user01 -Properties Department |
    Select-Object Name, SamAccountName, DistinguishedName, Department
```

sAMAccountName인 `user01`로 객체를 찾은 뒤 예상한 OU와 부서 값이 맞는지 확인한다. 조회 실패만으로 삭제를 단정하지 않는다. 접속한 도메인·DC, 입력한 식별자, 읽기 권한을 먼저 확인한다. [Get-ADUser](https://learn.microsoft.com/en-us/powershell/module/activedirectory/get-aduser)

## 파티션·경로·식별자를 분리한다

원본 AD LDS 문서는 구성·스키마·애플리케이션 파티션을 나눠 관리한다. 이 구분을 AD DS와 비교하면, 같은 LDAP 도구를 사용하더라도 접속한 서비스와 Naming Context를 먼저 확인해야 한다는 점이 드러난다.

| 데이터 범위 | AD DS에서의 의미 | AD LDS와 비교할 지점 |
|---|---|---|
| Schema | Forest의 클래스·속성 정의 | LDS 복제 집합은 AD DS Forest 스키마와 별개 |
| Configuration | Forest 구성·Site 등 | LDS 인스턴스·복제 집합의 구성 메타데이터 |
| Domain | 도메인 계정·그룹 등 | LDS에는 AD DS 도메인 대신 앱용 파티션 설계 |
| Application | DNS 등 선택 데이터의 파티션 | 앱 데이터의 DN·복제 대상을 별도로 정함 |

이 표는 데이터 범위를 비교하며 AD LDS가 도메인 인증·GPO·GC를 제공한다는 뜻은 아니다. 원본의 LDS 스키마 가져오기 화면도 AD DS Forest에 같은 변경을 적용했다는 증적으로 사용하지 않는다.

| 값 | 용도 | 변경 시 주의 |
|---|---|---|
| DN | 현재 계층 위치를 포함한 이름 | 이동·이름 변경 시 바뀜. Base DN 밖으로 이동하면 앱 검색에서 사라질 수 있음 |
| objectGUID | 디렉터리 객체 자체의 식별 | 같은 Forest 내 동일 객체의 이동·이름 변경에도 유지. 삭제 후 재생성은 새 객체 |
| objectSID | 보안 주체의 권한 식별 | 같은 이름으로 재생성해도 기존 ACL의 SID를 자동 승계하지 않음 |
| sAMAccountName·UPN | 로그인·계정 찾기 | 영구 식별자로 가정하지 않고 이름 변경·접미사·앱 규칙 검토 |
| displayName | 화면 표시 | 중복 가능성을 고려하고 연동 키로 단정하지 않음 |

일반 OU 이동과 도메인 간 마이그레이션은 같은 변경이 아니다. GUID·SID 유지 여부를 이관 도구·범위까지 확인한다. [객체 이름과 GUID](https://learn.microsoft.com/en-us/windows/win32/ad/object-names-and-identities), [SID와 계정 재생성](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-identifiers)

## 이동 전후의 관찰 대상을 정한다

앞의 Get-ADUser는 객체 속성 조회다. 앱이 쓰는 LDAP Bind·Search의 성공을 대신하지 않는다. 대상 DC를 고정한 다음 예제는 AD 모듈과 해당 객체 읽기 권한으로 실행한다.

```powershell
Get-ADRootDSE -Server DC01 |
    Select-Object defaultNamingContext, configurationNamingContext, schemaNamingContext
Get-ADUser -Server DC01 -Identity user01 -Properties UserPrincipalName, Department |
    Select-Object ObjectGUID, SID, DistinguishedName, SamAccountName, UserPrincipalName, Department
```

Naming Context와 객체 위치·식별자를 기록한다. 다른 DC에서 결과가 다르면 입력값·권한과 복제 시점을 먼저 확인한다. 이동 테스트는 승인된 테스트 객체로 수행하고 앱이 실제 사용하는 Base DN·Scope·Filter·읽기 권한에서도 결과를 비교한다. [Get-ADRootDSE](https://learn.microsoft.com/en-us/powershell/module/activedirectory/get-adrootdse), [Get-ADUser](https://learn.microsoft.com/en-us/powershell/module/activedirectory/get-aduser)

| 증상 | 확인 순서 | 변경·원복 판단 |
|---|---|---|
| 관리 콘솔에는 있으나 앱 검색 실패 | 서비스·포트 → Naming Context → Base DN·Scope → Filter → ACL | 검색 범위를 무조건 전체로 넓히지 않고 요구한 객체만 포함 |
| 이동 후 DN 기반 연동 실패 | 변경 DN·앱 저장 참조·GPO·ACL | 앱 참조 수정 또는 기록한 위치 복귀 후 재검증 |
| 같은 이름 재생성 후 권한 실패 | 이전·현재 SID와 리소스 ACL | 이름 일치만으로 기존 권한을 일괄 부여하지 않음 |
| 속성이 안 보임 | 조회 속성·읽기 권한·스키마 정의·도구 표시 | 스키마 확장 전에 원인 확정 |

## 스키마 확장은 별도 변경으로 다룬다

사용자 속성 값을 바꾸는 일과 새로운 속성 정의를 추가하는 일은 영향 범위가 다르다. AD DS 스키마 추가는 Forest 전체에 복제되고 일반 객체처럼 삭제해 되돌릴 수 없다. 비활성화가 가능한 경우에도 기존 데이터와 앱 영향은 남는다. 원본 LDS 문서의 변경·롤백 요구는 대상 복제 집합과 지원되는 복구 방법을 확인하는 변경 계획으로 구체화한다. 요구 속성·구문·다중값·고유 OID·명명·복제·호환성·백업을 테스트하고 가져오기 성공과 앱 성공을 따로 판정한다. [스키마 구현과 되돌릴 수 없는 추가](https://learn.microsoft.com/en-us/windows/win32/ad/schema-implementation)

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.

[연재 목차]({% post_url 2026-08-13-ad-series-guide %})
