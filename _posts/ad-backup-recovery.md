---
layout: single
title: "AD 22. DC 이중화·백업·복구는 무엇이 다른가"
categories: [Active Directory]
tags: [AD, Backup, System State, Forest Recovery]
toc: true
published: false
---

DC를 여러 대 운영해도 백업은 필요하다. 한 DC가 멈췄을 때 다른 DC가 서비스를 제공하는 것과, 잘못된 변경 이전의 상태를 되찾는 것은 다른 문제다.

복제는 같은 명부를 여러 곳에서 유지하는 작업이고, 백업은 특정 시점의 명부를 따로 보관하는 작업에 가깝다. 잘못된 삭제·변경이 다른 DC에 복제될 수 있으므로 복제만으로 과거 상태를 복구할 수 있다고 가정하지 않는다.

## 세 목적을 나눈다

| 구분 | 목적 | 확인할 질문 |
|---|---|---|
| 이중화 | 일부 서버 장애 중 서비스 지속 | 남은 DC로 업무가 가능한가? |
| 백업 | 복구에 사용할 시점·데이터 확보 | 필요한 구성과 복원 경로가 있는가? |
| 복구 계획 | 영향에 맞는 복원·서비스 재개 | 무엇을 어떤 순서로 복구하고 검증하는가? |

단일 DC 장애, 객체 삭제, Forest 전체 장애는 같은 절차로 처리하지 않는다. Microsoft Forest Recovery 가이드는 환경에 맞게 조정할 복구 계획의 출발점이다. [Microsoft Forest Recovery](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-guide)

## 백업 성공과 복원 가능성은 다르다

System State 백업과 전체 서버 복구에 필요한 백업의 범위는 구분한다. 백업 보관 기간, 무결성, 자격 증명, 복구에 사용할 OS·서버·도구와 격리 환경을 준비해야 한다.

계획에는 복구 대상 DC와 순서, DNS·시간 의존성, 역할 확인, 애플리케이션 검증, 연락·승인 체계를 기록한다. 복구 완료는 AD 서비스 시작만이 아니라 사용자 인증·정책·업무 시스템 동작까지 확인한 상태로 정의한다.

## 스냅샷을 백업의 대체물로 단정하지 않는다

가상 DC에는 VM-Generation ID 등 지원 조건에 따른 보호 장치가 있지만 모든 복원 방식이 동일하게 보호되는 것은 아니다. “스냅샷은 항상 안전하다”와 “스냅샷을 복원하면 항상 USN Rollback이다” 모두 조건을 빠뜨린 설명이다. 하이퍼바이저·게스트 지원과 복원 경로를 확인한다. [Microsoft 가상 DC 자료](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-virtualization)

## krbtgt 재설정은 즉시 두 번 실행하는 절차가 아니다

공식 Forest Recovery 절차는 두 번의 암호 재설정 사이 기본 10시간을 기다리도록 설명한다. 티켓 최대 수명 정책을 바꿨다면 대기 시간도 그 설정을 반영해야 한다. 원본의 “2회 연속 변경”을 즉시 실행하라는 의미로 옮기지 않는다. [Microsoft krbtgt 재설정](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-reset-the-krbtgt-password)

이 글은 복구 목적과 판단 기준을 정리한 글이다. 실제 환경의 복구 Runbook이나 복원 성공 증적을 제시하지 않는다. 실행 절차는 환경에 맞춰 검증하고 격리된 복구 훈련에서 확인해야 한다.

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.
