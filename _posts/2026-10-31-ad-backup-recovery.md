---
layout: single
title: "AD 22. 백업·복구 설계: 장애 범위·격리·복구 지점과 업무 검증"
categories: [Active Directory]
tags: [AD, Backup, System State, Forest Recovery]
toc: false
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

## 원본 복구·전환 표를 장애 범위로 나눈다

| 상황 | 복구 판단의 출발점 | 완료 증거 |
|---|---|---|
| 단일 DC 하드웨어·OS 장애 | 남은 DC·DNS·역할·Site 가용성과 원인 | 대체 경로의 인증·정책·복제와 재배포 DC 검증 |
| 객체 삭제·잘못된 속성 | 영향 객체·복제·복구 기능·유효 백업 | 속성·멤버십·ACL·앱 참조까지 확인 |
| SYSVOL·GPO 손상 | AD 정보와 파일·각 DC 상태·정상 기준 | 정책 파일·복제·클라이언트 실제 설정 |
| Forest 전체 장애·침해 | 격리·신뢰 가능한 백업·오염 범위·복구 책임 | 도메인별 복구와 안전한 업무 재개 |
| 계획 이관·Cut-over 실패 | 사전 Go/No-Go·변경분·새 데이터·복귀 가능성 | 이전·신규 업무 데이터의 정합성과 접근 |

이중화 DC를 다시 만드는 작업과 과거 데이터를 되살리는 작업을 같은 절차로 처리하지 않는다. 침해가 의심되면 오염된 DC를 그대로 기준으로 삼지 않고 조사·격리·복구 지점의 신뢰를 우선 판단한다. [Forest 복구 계획의 목적과 범위](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-guide)

## 백업 산출물의 복원 범위를 적는다

| 산출물 | 목적 | 별도로 필요한 것 |
|---|---|---|
| System State | DC의 디렉터리 관련 시스템 상태 복구 | 지원되는 OS·백업·복구 방식과 절차 |
| 전체 서버·BMR 백업 | OS·필수 볼륨 등 서버 복구 준비 | 하드웨어·드라이버·플랫폼·복구 매체 조건 |
| GPO 백업·설정 보고서 | 정책 설정 복원·비교 | 링크·필터·권한·파일 의존성 검증 |
| DNS 레코드 CSV·진단 내보내기 | 목록·변경 비교·문제 조사 | 영역 보안·AD/서버 복구 백업 |
| VM 체크포인트 | 지원 조건에서 특정 가상화 작업 | 독립 보관·오염 전 복구 지점·AD 지원 복원 경로 |

System State와 BMR은 같은 이름의 백업이 아니다. 작업 성공 로그뿐 아니라 백업 내용·보관 위치·읽기 자격 증명·암호화 키·복원 경로·유효 복구 시점을 인수한다. [System State 백업 범위와 절차](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-backing-up-system-state)

## 격리 복구 연습의 인수 항목

원본의 Forest Recovery와 복구 점검표를 다음 결과물로 연결한다. 아래는 순서를 결정할 체크포인트이며 현장 실행 명령을 대신하지 않는다.

1. 장애 유형과 복구 선언·책임자를 정하고 업무·관리망에서 복구 환경을 격리한다.
2. 백업 생성 시각·유효성·오염 여부·지원 조건을 확인해 도메인별 기준 복구 대상을 선정한다.
3. DNS·시간·역할·SYSVOL 등 의존성과 도메인 복구·다른 DC 재배포 순서를 환경별 계획에 맞춰 수행한다.
4. 자격 증명·신뢰·침해 경로 정리와 안전한 재연결 조건을 검증한다.
5. 인증·이름 확인·정책·그룹/리소스 접근·업무 앱을 대표 사용자·장비로 확인한다.

기록에는 목표 RPO/RTO와 실제 측정값을 구분한다. 이번 문서 작성 과정에서는 복원 훈련을 수행하지 않아 복구 시간이나 성공률을 만들지 않았다.

원복은 복구 작업을 역순으로 실행하는 뜻이 아니다. Cut-over 이후 새로 쓰인 계정·권한·앱 데이터를 어디까지 되돌릴지, 중복·유실·접근권 문제를 어떻게 확인할지 정한다. 백업이 오래됐거나 신뢰할 수 없는 경우에는 복구 확대 전에 의사결정을 다시 받는다.

## 스냅샷을 백업의 대체물로 단정하지 않는다

가상 DC에는 VM-Generation ID 등 지원 조건에 따른 보호 장치가 있지만 모든 복원 방식이 동일하게 보호되는 것은 아니다. “스냅샷은 항상 안전하다”와 “스냅샷을 복원하면 항상 USN Rollback이다” 모두 조건을 빠뜨린 설명이다. 하이퍼바이저·게스트 지원과 복원 경로를 확인한다. [Microsoft 가상 DC 자료](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-virtualization)

## krbtgt 재설정은 즉시 두 번 실행하는 절차가 아니다

공식 Forest Recovery 절차는 두 번의 암호 재설정 사이 기본 10시간을 기다리도록 설명한다. 티켓 최대 수명 정책을 바꿨다면 대기 시간도 그 설정을 반영해야 한다. 원본의 “2회 연속 변경”을 즉시 실행하라는 의미로 옮기지 않는다. [Microsoft krbtgt 재설정](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-reset-the-krbtgt-password)

이 글은 복구 목적과 판단 기준을 정리한 글이다. 실제 환경의 복구 Runbook이나 복원 성공 증적을 제시하지 않는다. 실행 절차는 환경에 맞춰 검증하고 격리된 복구 훈련에서 확인해야 한다.

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.

[연재 목차]({% post_url 2026-08-13-ad-series-guide %})
