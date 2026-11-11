---
layout: single
title: "AD 10. OU·위임 설계: 관리 그룹·상속·로그온 경로 검증하기"
categories: [Active Directory]
tags: [AD, OU, Delegation, Least Privilege]
toc: true
published: false
---

조직도를 그대로 OU로 만들면 부서 이동이나 개편 때마다 객체 이동과 정책 영향도 검토가 필요해진다. OU는 보기 좋은 분류보다 관리 책임과 정책 적용 범위를 기준으로 설계하는 것이 중요하다.

책장을 분야별로 나누는 것과 열쇠를 누구에게 줄지 정하는 것은 다른 일이다. OU를 만들 때도 보관할 객체, 적용 정책, 관리자를 함께 정해야 한다. [Microsoft OU 설계 개념](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/reviewing-ou-design-concepts)

## 예제 구조의 목적

```text
ad.example.com
├─ Users          사용자 계정 관리
├─ Workstations   사용자 PC 정책
└─ Servers        멤버 서버 정책
```

이는 단순화한 예제다. DC 전용 OU와 기본 컨테이너를 없애거나 모든 객체를 이 구조로 이동하라는 의미는 아니다. 하위 OU가 필요하면 담당자와 정책의 차이를 근거로 추가한다.

## 위임은 대상과 작업을 함께 제한한다

예를 들어 지원 담당자가 일반 사용자 암호를 재설정해야 한다고 해서 도메인 관리자 권한을 줄 필요는 없다. 특정 OU에 대해 필요한 작업을 그룹에 위임할 수 있다. 위임은 상위 컨테이너 아래의 객체에 영향을 줄 수 있으므로 상속 범위를 확인한다. [Microsoft 관리 위임](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/delegation-control-wizard)

| 결정 항목 | 설명용 예제 |
|---|---|
| 대상 | 일반 사용자 OU |
| 위임 그룹 | `GG_User_Support` |
| 허용 작업 | 일반 사용자 암호 재설정 |
| 제외 | 관리자 계정, 다른 OU, 그룹 권한 변경 |

## 허용과 거부를 모두 검증한다

테스트 계정으로 대상 OU의 일반 사용자 암호를 재설정할 수 있는지 확인하고, 대상 밖 계정과 다른 관리 작업은 거부되는지 확인한다. 관리자 권한이 함께 있는 계정으로 시험하면 위임이 과도하거나 부족한지 놓치기 쉽다.

고권한 보호 계정은 일반 객체와 다른 ACL 보호가 적용될 수 있다. 위임 테스트에는 일반 테스트 객체를 사용하고, 보호 계정의 동작을 일반 OU 위임과 혼동하지 않는다. [Microsoft 보호 계정·그룹](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/appendix-c--protected-accounts-and-groups-in-active-directory)

## 원본 위임 매트릭스를 작업별로 좁힌다

원본 구축 문서의 자산·관리자 분류는 책임을 나누는 출발점이다. OU 분리는 디렉터리의 논리 관리 구조이며 물리·침해 격리를 자동 보장하지 않는다. “상위 관리자이므로 모든 하위 서버에 로그온 허용”이라는 표로 옮기지 않고, 위임 권한과 자격 증명이 노출되는 로그온 경로를 나눠 정한다.

| 대상·그룹 예제 | 허용할 작업 | 분리해서 승인할 작업·검증 |
|---|---|---|
| Users / `GG_User_Support` | 일반 사용자 암호 재설정 | 관리자·서비스 계정 제외, 그룹 멤버십 변경 거부 |
| Workstations / `GG_PC_Operators` | 지정 단말 객체 관리 | 대상 밖 객체·서버 OU 수정 거부 |
| Servers / `GG_Server_Operators` | 승인된 서버 객체 작업 | GPO 편집·링크와 서버 로그온 권한을 별도 통제 |
| 서비스 계정 영역 | 승인된 계정 수명주기 | SPN·암호 조회 주체·서비스 권한은 별도 심사 |
| 고권한 계정·DC | 별도 관리 경로와 책임 | 일반 지원 그룹 위임에서 제외, 전용 관리 단말 검증 |

GPO 연결 권한을 주는 일은 많은 장비의 정책에 영향을 줄 수 있다. 사용자 속성 편집 위임과 같은 영향도로 취급하지 않는다. 검토 주기는 원본의 분기·반기 예제를 복사하기보다 권한 영향과 조직 변경 주기에 맞춰 정한다.

원본의 OU 3~4단계는 관리 편의를 위한 선택이지 기술 한계나 정책 처리 속도 보장값이 아니다. 연동 앱의 DN 길이·검색 경로 제한도 함께 확인한다. [OU 깊이와 앱 제약](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/reviewing-ou-design-concepts)

## ACL 조회와 실제 위임 테스트를 연결한다

ADUC의 고급 기능을 켠 뒤 대상 OU의 Security → Advanced에서 주체·작업·적용 객체 종류·상속 범위를 읽는다. 위임 작업에는 해당 ACL 변경 권한이 필요하고, 결과 확인은 위임받은 테스트 계정으로 수행한다. AD 모듈의 OU 조회는 위치 확인에 쓰며 ACL의 유효 권한을 대신하지 않는다.

| 테스트 | 기대 관찰 | 실패 해석 |
|---|---|---|
| 대상 일반 객체의 승인 작업 | 성공 | 상속·객체 종류·속성 권한 확인 |
| 같은 OU의 비승인 작업 | 거부 | 성공하면 위임·중첩 그룹 과다 여부 조사 |
| 다른 OU의 같은 작업 | 거부 | 상위 ACL·다른 그룹 권한 확인 |
| 보호 계정 | 일반 위임과 별도 판정 | AdminSDHolder 보호·상속 상태를 무작정 해제하지 않음 |

고권한 그룹에서 빠진 계정도 보호 이력과 ACL 상태를 확인해야 한다. 상속을 다시 켜는 변경 역시 영향 검토 대상이다. [보호 계정과 AdminSDHolder](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/appendix-c--protected-accounts-and-groups-in-active-directory)

## OU 이동은 변경 작업이다

객체 위치를 바꾸면 적용 GPO와 상속 ACL, DN을 참조하는 연동이 영향을 받을 수 있다. 이동 전후 정책·위임·애플리케이션을 비교하고 원복할 위치도 기록한다. 관리 인수인계에는 OU 이름뿐 아니라 목적·담당 그룹·허용 작업·해제 절차를 남긴다.

권한 위임 변경 전에는 대상 DN과 ACL·그룹 멤버십을 기록한다. 원복은 이번에 추가한 권한만 식별해 회수하고 기존 업무 권한을 재검증한다. OU 전체 ACL 초기화나 삭제는 일반적인 위임 해제 절차로 사용하지 않는다.

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.

[연재 목차]({% post_url 2026-11-10-ad-series-guide %})
