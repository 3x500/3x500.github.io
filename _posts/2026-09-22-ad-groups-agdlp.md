---
layout: single
title: "AD 11. AGDLP 운영: 중첩 권한·직무 이동·실제 접근 검증"
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

## 원본 Mover 절차를 회수·부여 표로 만든다

원본 구축 가이드의 역할→리소스 그룹 모델과 보안 문서의 직무 이동 절차를 연결한다. 아래는 인사부에서 재무부로 이동하는 가상 사용자 예제다. 실제 변경 이력이나 수행 결과가 아니다.

| 순서 | 기록·작업 | 검증 |
|---|---|---|
| 변경 인지 | 이전·새 직무와 적용 시점·승인자 | 임시 겸직·예외 만료 여부 |
| 역할 비교 | 이전 역할·중첩 그룹·직접 ACL·앱 권한 | 회수 대상과 유지 사유를 구분 |
| 회수 | `GG_HR` 등 이전 역할 연결 제거 | `DL_HR_Read`로 이어지는 다른 중첩 경로·직접 권한 잔존 확인 |
| 부여 | 승인한 `GG_Finance` 역할 연결 | 필요한 `DL_Finance_Read`와 리소스 ACL 연결 |
| 재검토 | 새 세션으로 허용·거부 확인 | 이전 자료 거부·새 자료 읽기·비승인 수정 거부 |
| 증적 | 승인·변경 전후·접근 결과·예외 기한 | 리소스 오너 확인과 감사 기록 보관 |

권한 회수와 부여의 순서는 업무 연속성·긴급 회수 요구에 따라 승인해 정한다. 이전 권한을 무기한 남기는 방식으로 전환 공백을 해결하지 않는다. AD 그룹 밖의 메일·앱·로컬 그룹·직접 ACL도 별도 점검 대상이다.

## 그룹 범위와 최종 토큰은 서로 다른 검사다

| 범위 | 멤버 범위의 핵심 | 권한 부여 범위 |
|---|---|---|
| Global | 같은 도메인의 계정·Global 그룹 | 신뢰·권한 조건을 만족하는 리소스 |
| Universal | 같은 Forest의 계정·Global·Universal 그룹 | 신뢰·권한 조건을 만족하는 리소스 |
| Domain Local | 신뢰되는 도메인의 계정·Global·Universal 및 같은 도메인의 Domain Local 그룹 | 해당 도메인의 리소스 |

표는 대표적인 중첩 규칙이며 계정 종류·신뢰 범위와 상세 제약은 실제 설계에서 대조한다. 다중 도메인에서 역할을 집약하려면 Universal 계층의 필요성과 GC 복제 영향을 검토한다. [그룹 범위 규칙](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-groups#group-scope)

직접 멤버 조회와 `-Recursive` 조회를 나눠 저장한다. 후자는 중간 그룹 경로를 펼친 구조도와 같지 않다. 사용자가 현재 가진 토큰은 AD의 최신 멤버십과 다를 수 있다. 해당 사용자 세션에서 `whoami /groups`를 읽고, 새 로그온·세션 조건을 맞춰 실제 리소스 접근과 연결한다. 명령 출력에는 SID·그룹명이 있으므로 그대로 공개하지 않는다.

## 최종 접근까지 확인한다

그룹 구성이 맞아도 공유·NTFS 권한, 거부 항목, 현재 세션의 토큰에 따라 접근 결과가 달라질 수 있다. 실제 접근 계정과 경로를 정하고 읽기·쓰기·삭제처럼 필요한 작업별로 확인한다. [Microsoft 접근 제어](https://learn.microsoft.com/en-us/windows/security/identity-protection/access-control/access-control)

부서 이동 시 새 그룹에 추가하는 것만큼 기존 권한 회수도 중요하다. 그룹 소유자, 부여 사유, 정기 검토 주기를 기록해야 시간이 지나도 역할 그룹의 의미가 유지된다.

변경 실패 시 기록한 멤버십·ACL에서 이번 변경분을 복구하되, 보안상 회수한 권한을 자동으로 재부여하지 않는다. 리소스 오너의 판단으로 접근을 복원하고 읽기·쓰기·삭제와 이전 역할 거부를 다시 시험한다. 그룹 삭제·재생성은 SID가 달라질 수 있어 일반 원복으로 쓰지 않는다.

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.

[연재 목차]({% post_url 2026-08-13-ad-series-guide %})
