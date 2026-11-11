---
layout: single
title: "AD 18. Windows LAPS 설계: 저장·회전·조회·복호화 권한 검증"
categories: [Active Directory]
tags: [AD, Windows LAPS, Local Administrator, DSRM]
toc: true
published: false
---

모든 PC의 로컬 관리자 암호가 같으면 한 장비에서 노출된 암호가 다른 장비 접근에도 사용될 수 있다. 장비별 암호를 다르게 관리하고 교체하는 작업을 자동화하는 기능이 Windows LAPS다.

모든 문에 같은 비상 열쇠를 쓰는 대신 문마다 다른 열쇠를 두고 보관·교체 권한을 관리하는 방식에 가깝다. Windows LAPS는 로컬 관리자 암호와, 지원 조건을 갖춘 DC의 DSRM 암호를 관리할 수 있다. 도메인 사용자 암호를 일괄 회전하는 기능은 아니다. [Microsoft Windows LAPS 개요](https://learn.microsoft.com/en-us/windows-server/identity/laps/laps-overview)

## 암호 회전보다 먼저 정할 것

| 항목 | 결정할 내용 |
|---|---|
| 관리 대상 | 어떤 로컬 계정 또는 DSRM 계정인가? |
| 저장소 | AD DS 또는 Microsoft Entra ID 중 어떤 경로인가? |
| 조회·복호화 권한 | 누가 어떤 장비의 암호를 확인할 수 있는가? |
| 사용 후 처리 | 암호 교체와 감사 기록을 어떻게 관리하는가? |

DSRM 암호는 Entra ID에 백업할 수 없다. AD DS 저장과 암호화를 사용해야 하며, 도메인 기능 수준 2016 이상과 필요한 업데이트를 갖춘 Windows Server 2019 이상 DC 등 지원 조건을 충족해야 한다. [Microsoft DSRM 지원 조건](https://learn.microsoft.com/en-us/windows-server/identity/laps/laps-concepts-overview#dsrm-password-support)

계정이 존재하는지, 정책이 그 계정을 올바르게 가리키는지 먼저 확인한다. 계정 자동 생성·관리는 OS와 기능에 따라 지원 조건이 다르므로 모든 버전에서 동일하게 동작한다고 가정하지 않는다.

## OS와 기능 수준을 함께 본다

Windows LAPS와 레거시 Microsoft LAPS는 구분한다. 장비의 지원 OS·업데이트와 정책 템플릿, 기존 정책의 공존 여부를 확인해야 한다.

AD에 저장하는 암호의 암호화는 도메인 기능 수준 2016 이상이라는 조건이 있다. DSRM 관리도 DC OS 등 추가 조건이 있으므로 “LAPS를 지원한다”는 말만으로 기능 전체를 사용할 수 있다고 판단하지 않는다. [Microsoft AD 기반 배포 조건](https://learn.microsoft.com/en-us/windows-server/identity/laps/laps-scenarios-windows-server-active-directory)

AD 저장 경로에서는 Windows LAPS용 스키마 준비, 관리 장비의 암호 속성 갱신 권한, 관리자의 조회·복호화 권한을 구분해 확인한다. 정책 링크만 만들고 이 준비를 생략하면 암호 백업이 실패할 수 있다.

## 원본 구조도를 AD 저장 경로에 맞춰 읽는다

![원본 보안 문서의 Windows LAPS 동작 설명도]({{ '/images/ad-deep-dive/c-012.png' | relative_url }})

그림은 로컬 암호 회전→디렉터리 백업→승인 조회를 설명한다. 하단의 Entra ID·Intune 문구는 가능한 별도 관리 시나리오이며 이 글의 AD 저장 실습이나 클라우드 운영 실적을 뜻하지 않는다. AD 저장 암호를 Intune 포털에서 직접 조회한다고 해석하지 않는다. DSRM은 AD 저장·암호화 조건으로 별도 검증한다.

| 준비 단계 | 필요한 권한·대상 | 확인·실패 분기 |
|---|---|---|
| 스키마 준비 | 승인된 Forest 스키마 변경 주체 | Windows LAPS 속성 정의와 복제. 일반 정책 원복처럼 제거하지 않음 |
| 장비 쓰기 | 대상 OU에서 관리 컴퓨터의 필요한 속성 쓰기 | 정책을 받아도 백업 거부되면 ACL·복제·도메인 연결 확인 |
| 읽기 위임 | 승인된 지원 그룹, 해당 장비 범위 | 암호 관련 읽기·상속·기존 확장 권한 검토 |
| 암호화·복호화 | DFL·암호화 정책·승인 복호화 주체 | 읽기 가능과 복호화 가능을 별도 시험 |
| 사용 후 처리 | 지원 OS·정책의 회전·세션 처리 | 긴급 접근과 서비스 의존성 영향 확인 |

원본 도입표의 스키마·위임·정책·감사를 각각 완료 조건으로 사용한다. 이미 넓은 확장 권한을 가진 주체가 암호를 읽을 수 있는지 조사해야 한다. 새 조회 그룹만 좁혀 만들었다고 기존 과다 권한까지 제거된 것은 아니다. [AD 기반 배포·권한 준비](https://learn.microsoft.com/en-us/windows-server/identity/laps/laps-scenarios-windows-server-active-directory)

## OS·저장소별 선택의 비용

레거시 LAPS와 Windows LAPS의 정책·속성·에이전트를 구분한다. 기존 정책 충돌과 전환 경로를 확인하고 파일럿에서만 바꾼다. 계정 자동 관리는 Windows 11 24H2·Windows Server 2025 이상 등 별도 조건이 있어, 구버전의 계정 생성까지 자동이라고 쓰지 않는다. [Windows LAPS 플랫폼·자동 계정 관리 조건](https://learn.microsoft.com/en-us/windows-server/identity/laps/laps-overview#windows-laps-supported-platforms)

저장소와 정책 경로를 결정하고 실제 장비의 유효 정책을 확인한다. Hybrid 가입이라는 이유만으로 AD와 Entra ID에 같은 암호를 동시에 백업하는 모델로 가정하지 않는다. [LAPS 저장과 배포 구조](https://learn.microsoft.com/en-us/windows-server/identity/laps/laps-concepts-overview)

## 적용 상태는 장비에서도 확인한다

정책 링크가 있다는 사실과 암호 백업 성공은 다른 확인이다. 장비의 `Microsoft-Windows-LAPS/Operational` 로그에서 처리 시작·완료, 적용 정책, 오류를 확인한다. 이 글은 실제 암호나 운영 로그 화면을 포함하지 않는다. [Microsoft LAPS 이벤트 로그](https://learn.microsoft.com/en-us/windows-server/identity/laps/laps-management-event-log)

처리 주기의 시작 10003, 성공 10004 또는 실패 10005와 중간 정책·백업·암호 변경 이벤트를 묶어 읽는다. 단일 이벤트만 보지 않고 같은 주기의 저장 대상·오류·완료를 확인한다. [LAPS 처리 주기 이벤트](https://learn.microsoft.com/en-us/windows-server/identity/laps/laps-management-event-log#policy-processing-cycle-start-and-end)

## 파일럿 완료 기준

암호가 교체·백업되고, 허용된 관리자는 조회할 수 있으며, 허용되지 않은 계정은 조회할 수 없어야 한다. 비상 접근 절차에서 실제로 사용할 수 있는지도 확인한다.

조회된 암호를 작업 기록이나 블로그 캡처에 남기지 않는다. 암호 조회 권한을 제한하지 않으면 장비별 암호를 분리한 효과가 줄어든다. 회전·백업·접근 통제를 함께 관리한다. [Microsoft LAPS 동작 구조](https://learn.microsoft.com/en-us/windows-server/identity/laps/laps-concepts-overview)

정책 원복이 이미 회전한 암호를 이전 공통 암호로 되돌린다고 가정하지 않는다. 변경 전 유효 정책·읽기/쓰기/복호화 권한·복구 접근을 기록하고, 백업 또는 긴급 접근 실패 시 확대를 중단한다. 파일럿의 정책·위임을 조정한 뒤 승인 조회·비승인 거부·재회전·백업을 다시 검증한다. 스키마 준비는 별도 변경으로 남긴다.

---

적용 기준: 업데이트된 Windows Server 2019 이상과 지원 Windows 11의 Windows LAPS. DFL·DSRM 조건은 본문 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.

[연재 목차]({% post_url 2026-08-13-ad-series-guide %})
