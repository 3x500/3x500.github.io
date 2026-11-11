---
layout: single
title: "AD 17. 관리자 보안: 관리 경로 분리와 비인가 로그온 탐지 파이프라인"
categories: [Active Directory]
tags: [AD, Least Privilege, Privileged Access, Audit]
toc: true
published: true
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

## 권한과 자격 증명이 지나가는 경로를 분리한다

원본의 Tier 0/1/2는 DC·서버·단말 관리 책임과 자격 증명 노출을 나누는 모델이다. 상위 관리 계정을 하위 장비에 모두 허용하는 의미로 옮기지 않는다. 현재 Microsoft Enterprise Access Model은 온프레미스 AD 밖의 클라우드·관리 인터페이스까지 포함하므로 기존 Tier 표와 같은 범위로 취급하지 않는다. [Enterprise Access Model](https://learn.microsoft.com/en-us/security/privileged-access-workstations/privileged-access-access-model)

```text
일반 업무 계정 → 업무 단말·메일·웹
서버 관리 계정 → 승인된 관리 단말·경로 → 지정 서버
AD 고권한 계정 → 더 신뢰할 수 있는 전용 관리 경로 → AD 관리 대상
```

관리할 대상보다 낮은 신뢰의 단말에 고권한 자격 증명을 노출하지 않도록 경로를 정한다. 계정 분리와 함께 대화형·RDP·네트워크·서비스 로그온 유형, 원격 관리 도구, 자격 증명 전달 방식을 검토한다. OU 이름과 그룹만으로 이 경로가 강제되지는 않는다. [보안 관리 호스트](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/implementing-secure-administrative-hosts)

## 원본 4단계 관제 흐름을 수집 범위와 연결한다

![원본 보안 문서의 관리자 로그온 탐지 4단계 설명용 파이프라인]({{ '/images/ad-deep-dive/c-003.png' | relative_url }})

이 그림은 탐지 설계 설명이며 실제 경고·차단 성공 화면이 아니다. 원본의 DC 로그 수집만으로 모든 단말의 관리자 로그온을 포괄한다고 설명하지 않는다. **4624는 로그온 세션이 생성된 대상 컴퓨터에 기록된다.** 서버·단말 로그온까지 탐지하려면 해당 대상의 감사·수집 범위도 포함한다. [이벤트 4624의 발생 위치](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4624)

| 단계 | 원본 구조를 반영한 설계 | 실패·누락을 드러낼 기준 |
|---|---|---|
| 상태 관리 | 수집 원본·채널별 마지막 처리 위치, 중복 억제 | 재시작·로그 순환·수집 지연·상태 손상 탐지 |
| 수집·정제 | 대상 컴퓨터, 계정 SID·도메인, 로그온 유형, 주소·시각 | 필드 누락을 정상·허용으로 처리하지 않음 |
| 교차 검증 | 고권한 그룹의 직접·중첩 관계와 승인 단말·작업 | 최신 허용 목록·예외 기한·DNS 변경 추적 |
| 경고·사후 처리 | 근거 이벤트·판정 이유·전송 결과·재시도 | 저장·전송 실패와 억제 건수를 별도 관찰 |

DNS 정·역방향 일치는 출발지 정보를 대조하는 보조 자료다. 단독으로 단말 신원을 확정하거나 경고를 억제할 신뢰 근거로 사용하지 않는다. NAT·VPN·프록시·누락 필드와 계정 이름 중복도 고려한다.

원본의 60초 쿨다운은 알림 조절 예제다. 계정·원본·규칙·사건을 구분하지 않은 전역 쿨다운은 서로 다른 사건을 숨길 수 있다. Record ID도 여러 컴퓨터·채널에 걸쳐 전역 고유값으로 쓰지 않는다. 억제한 원본 이벤트를 보존하고 조사할 수 있어야 한다.

## 탐지 검증은 정상·비정상·운영 실패를 포함한다

| 입력·상황 | 확인할 결과 |
|---|---|
| 허용 단말의 승인 작업 | 정상 분류의 근거와 원본 이벤트 보존 |
| 비허용 단말의 관리자 로그온 | 계정·대상·로그온 유형을 포함한 경고 |
| 다른 계정·다른 단말의 연속 사건 | 쿨다운으로 서로를 숨기지 않음 |
| DNS 실패·허용 목록 누락 | 판단 불가·수집 오류를 정상과 구분 |
| 재시작·로그 삭제·전송 실패 | 누락 구간·재처리·전송 상태를 추적 |
| DC 수집만 된 상태 | 서버·단말 관측 공백을 명시 |

특권 그룹 변경과 감사 변경은 변경 승인·수행자·대상·시각으로 상관분석한다. 이벤트 번호 한 개나 경고 카드만으로 침해를 확정하지 않는다.

스크립트·허용 목록·상태·대시보드 쓰기 권한을 분리하고 관리자만 판정 규칙을 변경할 수 있게 한다. 실행 정책만으로 보안 경계가 생기는 것은 아니다. 이 글은 관제 전체 코드나 배포 완료 증적을 새로 만들어 제시하지 않으며, 원본의 구조와 검증 요건을 설명한다.

## Protected Users와 암호 정책의 적용 조건

Protected Users는 관리자 계정에 추가할 수 있는 보호 수단이지만 기존 관리 경로의 호환성을 먼저 검사한다. 지원 OS·DFL·AES 준비를 확인하고 NTLM·자격 증명 캐시·위임·티켓 동작 제약이 도구와 앱에 미치는 영향을 파일럿에서 확인한다. 서비스·컴퓨터 계정을 일반 사용자처럼 일괄 넣거나 긴급 복구 계정까지 검증 없이 포함하지 않는다. [Protected Users 지원·제약](https://learn.microsoft.com/en-us/windows-server/security/credentials-protection-and-management/protected-users-security-group)

원본의 암호 만료 90일은 조직 설계안이다. Microsoft 공통 필수 주기나 모든 환경의 정답으로 쓰지 않는다. 공식 설명도 현대 완화 수단이 있는 Baseline과 그렇지 않은 환경의 조건을 구분한다. 현재 방어 수단·규정·침해 시 회수 절차를 검토하고 만료를 일괄 해제하는 권고로 바꾸지 않는다. [Maximum password age의 환경 조건](https://learn.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/maximum-password-age)

## 변경 완료 기준

필요한 작업은 성공하고, 대상 밖 작업은 거부되는지 확인한다. 계정 분리·그룹 정리·로그온 제한을 한 번에 전체 배포하지 않고 파일럿과 비상 접근 절차를 준비한다. 장애를 피하려고 예외를 늘릴 때도 사유와 종료 조건을 기록한다.

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.

[연재 목차]({% post_url 2026-08-13-ad-series-guide %})
