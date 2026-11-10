---
layout: single
title: "AD 03. 객체·속성·스키마·DN 이해하기"
categories: [Active Directory]
tags: [AD, Schema, LDAP, DN]
toc: true
published: false
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

## 스키마 확장은 별도 변경으로 다룬다

사용자 속성 값을 바꾸는 일과 새로운 속성 정의를 추가하는 일은 영향 범위가 다르다. 스키마 변경은 Forest 전체에 관계되므로 애플리케이션 요구, 중복 속성, 테스트·복구 전략을 검토해야 한다. 화면에서 속성이 보이지 않는다고 바로 스키마 확장을 결정하지 않는다.

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.
