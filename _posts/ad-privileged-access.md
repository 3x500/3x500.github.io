---
layout: single
title: "AD 17. 관리자 계정 분리와 최소 권한"
categories: [Active Directory]
tags: [AD, Least Privilege, Privileged Access, Audit]
toc: true
published: false
---

계정 생성·암호 재설정·서버 관리가 모두 필요하다고 모든 담당자에게 Domain Admin 권한을 주면 일하기는 편할 수 있다. 하지만 한 계정이 노출됐을 때 영향도 함께 커진다.

건물 전체의 마스터키 대신 맡은 구역의 열쇠를 주는 것이 최소 권한의 출발점이다. 어떤 작업을 누가 어느 장비에서 수행해야 하는지 정의한 뒤 권한을 부여한다. [Microsoft 최소 권한 관리 모델](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/implementing-least-privilege-administrative-models)

## 계정 이름만 나누면 끝나지 않는다

| 구분 | 운영 기준의 예 |
|---|---|
| 일반 계정 | 메일·웹·일반 업무 |
| 위임 관리 계정 | 지정 OU·서버의 허용 작업 |
| AD 고권한 계정 | 승인된 AD 관리 작업과 관리 단말 |

관리 계정을 분리했더라도 일반 사용자 PC에서 고권한 세션을 열고 계속 사용하면 자격 증명 노출 경로가 남는다. 관리 단말과 접속 경로, 로그온 허용 범위를 함께 관리한다. [Microsoft 보안 관리 호스트](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/implementing-secure-administrative-hosts)

## 권한을 줄이기 전에 업무를 분해한다

사용자 지원 업무라면 대상 OU의 암호 재설정 등 필요한 작업을 위임한다. 서버 운영은 해당 서버의 관리 권한을 별도로 정의한다. AD 전체 관리가 필요한 작업만 승인된 고권한 절차로 처리한다.

권한 회수도 함께 설계한다. 부서 이동 후 새 그룹만 추가하면 이전 권한이 쌓인다. 그룹 소유자·승인 사유·만료 또는 재검토 시점을 기록한다.

## 성공 로그온 하나로 판단하지 않는다

원본 업무 범위에는 비인가 단말의 관리자 로그온 탐지와 권한 변경 모니터링이 포함된다. 공개 글에서는 실제 단말·계정·경고 화면 대신 탐지 시 확인할 기준을 정리한다.

| 관찰 대상 | 함께 확인할 맥락 |
|---|---|
| 로그온 성공·실패 | 계정, 로그온 유형, 출발지, 시간 |
| 특권 그룹 변경 | 변경 대상, 수행자, 승인 작업 여부 |
| 감사 정책 변경·로그 삭제 | 정상 점검 여부와 다른 이벤트의 연관 |

이벤트가 남으려면 관련 감사 설정과 수집 체계가 필요하다. 단일 이벤트를 곧바로 침해로 단정하지 않고 변경 이력과 함께 해석한다. [Microsoft 모니터링 이벤트](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/appendix-l--events-to-monitor)

## 변경 완료 기준

필요한 작업은 성공하고, 대상 밖 작업은 거부되는지 확인한다. 계정 분리·그룹 정리·로그온 제한을 한 번에 전체 배포하지 않고 파일럿과 비상 접근 절차를 준비한다. 장애를 피하려고 예외를 늘릴 때도 사유와 종료 조건을 기록한다.

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.
