---
layout: single
title: "AD 12. GPO 적용 순서와 적용 범위"
categories: [Active Directory]
tags: [AD, GPO, Group Policy]
toc: true
published: false
---

GPO를 만들었다고 정책이 모든 PC에 적용되지는 않는다. 설정 내용과 적용 대상을 따로 확인해야 한다. GPO는 규정집, 링크는 어느 구역에 그 규정을 적용할지 정하는 연결로 생각하면 쉽다.

GPO는 Site·Domain·OU에 연결할 수 있다. 보안 그룹은 GPO를 직접 연결하는 컨테이너가 아니라 적용 대상 필터링 등에 사용한다. [Microsoft GPO 범위](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-policy/group-policy-scope)

## 기본 처리 순서

```text
Local → Site → Domain → 상위 OU → 하위 OU
```

일반적인 설정 충돌에서는 뒤에서 처리되는 정책이 앞선 값을 덮어쓸 수 있다. 같은 컨테이너의 여러 링크에는 링크 순서도 영향을 준다. 하지만 “가장 아래 OU가 언제나 이긴다”로 외우면 강제 적용 같은 예외를 놓친다. [Microsoft GPO 처리 방식](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-policy/group-policy-processing)

같은 컨테이너에서는 링크 순서 1이 가장 높은 우선순위를 가진다. 상속 차단은 상위의 일반 링크에서 내려오는 정책을 차단하지만, 강제 적용된 상위 링크까지 차단하지는 않는다. 강제 적용된 상위 링크의 충돌 설정은 하위 설정으로 덮어쓸 수 없다. 강제 적용이 보안·WMI 필터나 필요한 권한을 무시한다는 뜻은 아니다.

## 적용 여부를 바꾸는 조건

| 조건 | 확인할 내용 |
|---|---|
| 링크 위치·활성 상태 | 대상 객체의 경로에 적용되는 링크인가? |
| 사용자·컴퓨터 설정 | 바꾸려는 항목이 어느 쪽 설정인가? |
| 보안 필터링 | 필요한 읽기·적용 권한이 있는가? |
| WMI 필터 | 장비가 필터 조건을 충족하는가? |
| 상속 차단·강제 적용 | 기본 처리 순서에 예외가 있는가? |

사용자 설정은 사용자 객체, 컴퓨터 설정은 컴퓨터 객체의 위치가 기본 기준이다. 루프백 처리를 사용하면 사용자 정책 적용이 달라질 수 있으므로 사용 목적과 범위를 별도로 기록한다.

## 예외 정책의 범위를 작게 잡는다

공용 PC에서 사용자 환경을 제한하려는 정책과 서버 보안 정책을 하나의 GPO에 섞으면 영향 추적이 어려워진다. 대상과 변경 이유를 기준으로 나누되, 설정 하나마다 GPO를 만드는 식의 과도한 분리도 피한다. [Microsoft GPO 개요](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-policy/group-policy-overview)

새 GPO는 파일럿 OU에서 대상 장비로 확인한다. 설정값뿐 아니라 적용 대상과 제외 대상이 의도대로 구분되는지 봐야 한다. 암호 정책처럼 도메인 계정에 대한 별도 적용 규칙이 있는 항목은 일반 OU 설정과 동일하게 설명하지 않는다.

변경 기록에는 GPO 이름, 링크 위치, 필터, 변경한 설정, 기대 동작, 원복 방법을 남긴다. 적용 결과 확인은 다음 글에서 다룬다.

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.
