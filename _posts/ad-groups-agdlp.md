---
layout: single
title: "AD 11. 보안 그룹과 AGDLP로 권한 관리하기"
categories: [Active Directory]
tags: [AD, Security Group, AGDLP, ACL]
toc: true
published: false
---

파일마다 사용자 계정을 직접 등록하면 누가 어떤 이유로 접근하는지 추적하기 어렵다. 인사 이동 때 이전 권한을 회수하는 작업도 빠지기 쉽다. 그룹은 계정과 리소스 권한 사이에 관리 단위를 만든다.

직원 이름을 모든 출입문에 적는 대신 역할별 출입증을 관리하는 방식에 가깝다. 다만 그룹에 넣었다는 사실만으로 권한이 생기지는 않는다. 해당 그룹이 실제 리소스의 ACL과 연결돼야 한다. [Microsoft 보안 그룹](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-groups)

## AGDLP의 흐름

```text
Accounts → Global group → Domain Local group → Permissions
user01   → GG_Finance   → DL_Finance_Read   → 폴더 읽기
```

예제에서 Global 그룹은 업무 역할을, Domain Local 그룹은 리소스 접근 단위를 표현한다. 사람의 소속이 바뀌면 역할 그룹 멤버십을 바꾸고, 폴더 권한 구조는 유지할 수 있게 설계한다.

| 그룹 범위 | 설계할 때 볼 내용 |
|---|---|
| Global | 동일 도메인의 역할·계정 묶음 |
| Domain Local | 해당 도메인의 리소스 권한 부여 |
| Universal | 여러 도메인을 묶는 설계가 필요할 때 멤버십·복제 영향 검토 |

그룹 범위에는 멤버로 넣을 수 있는 대상과 권한을 부여할 수 있는 범위의 제약이 있다. 여러 도메인이면 같은 패턴을 기계적으로 적용하지 말고 범위 규칙을 확인한다.

## 멤버십 조회 예제

Active Directory 모듈이 있는 관리 PC에서 해당 그룹을 읽을 권한으로 실행한다. 이름은 가상 값이며 결과는 실제 환경에서 확인해야 한다.

```powershell
Get-ADGroupMember -Identity GG_Finance
Get-ADGroupMember -Identity DL_Finance_Read
```

직접 멤버십과 중첩 구조를 먼저 확인한다. `-Recursive`는 하위 구성원을 확인하는 데 유용하지만 중간 그룹 구조 파악을 대신하지 않는다. [Get-ADGroupMember](https://learn.microsoft.com/en-us/powershell/module/activedirectory/get-adgroupmember)

## 최종 접근까지 확인한다

그룹 구성이 맞아도 공유·NTFS 권한, 거부 항목, 현재 세션의 토큰에 따라 접근 결과가 달라질 수 있다. 실제 접근 계정과 경로를 정하고 읽기·쓰기·삭제처럼 필요한 작업별로 확인한다. [Microsoft 접근 제어](https://learn.microsoft.com/en-us/windows/security/identity-protection/access-control/access-control)

부서 이동 시 새 그룹에 추가하는 것만큼 기존 권한 회수도 중요하다. 그룹 소유자, 부여 사유, 정기 검토 주기를 기록해야 시간이 지나도 역할 그룹의 의미가 유지된다.

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.
