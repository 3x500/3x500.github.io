---
layout: single
title: "AD 10. OU 설계와 관리 권한 위임"
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

## OU 이동은 변경 작업이다

객체 위치를 바꾸면 적용 GPO와 상속 ACL, DN을 참조하는 연동이 영향을 받을 수 있다. 이동 전후 정책·위임·애플리케이션을 비교하고 원복할 위치도 기록한다. 관리 인수인계에는 OU 이름뿐 아니라 목적·담당 그룹·허용 작업·해제 절차를 남긴다.

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.
