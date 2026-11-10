---
layout: single
title: "AD 02. Forest·Domain·OU는 어떻게 다른가"
categories: [Active Directory]
tags: [AD, Forest, Domain, OU]
toc: true
published: false
---

Forest·Domain·OU는 모두 AD 구조를 설명하지만, 같은 목적의 구획은 아니다. 큰 상자 안에 작은 상자가 들어간 모습으로 시작하되, 각각 무엇을 공유하고 무엇을 관리하는지 구분해야 한다.

## 세 구조의 역할

| 구조 | 이해할 기준 | 예제 |
|---|---|---|
| Forest | 스키마·구성·GC 등을 공유하는 최상위 논리 구조 | 하나의 Forest |
| Domain | 계정·데이터 복제·관리의 도메인 단위 | `ad.example.com` |
| OU | 객체 관리 위임과 GPO 적용을 위한 컨테이너 | `Workstations`, `Servers` |

Forest에는 하나 이상의 Domain이 있고, Domain 안에 OU를 구성한다. 이 논리 구조는 서버를 몇 대 배치하고 어느 지점에 둘지와 구분된다. 같은 도메인에 여러 지역의 DC를 둘 수 있다. [Microsoft 논리 모델](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/understanding-the-active-directory-logical-model)

## 부서가 다르다고 Domain을 나눌 필요는 없다

인사팀과 재무팀의 PC 정책이 다르면 OU와 GPO로 관리 범위를 나눌 수 있다. 팀별 계정 관리 업무도 필요한 권한을 위임해 처리할 수 있다. 부서가 늘었다는 이유만으로 도메인을 추가하면 DNS·복제·관리 절차도 함께 복잡해진다.

설계할 때는 먼저 “다르게 적용할 정책은 무엇인가”, “누구에게 어느 객체의 관리를 맡길 것인가”를 묻는다. OU로 충족되는 요구인지 확인한 뒤 더 큰 구조 변경을 검토하는 편이 관리하기 쉽다. [Microsoft OU 설계 개념](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/reviewing-ou-design-concepts)

## OU를 보안 격리 경계로 보지 않는다

OU는 관리 범위를 구분하는 데 유용하지만, 서로 독립적인 AD 환경을 만드는 수단은 아니다. 같은 Forest 안에서 도메인을 나눴다는 이유만으로 서로 완전히 격리됐다고 보아서도 안 된다. Forest 수준의 보안 경계와 OU의 관리 경계는 구분해야 한다. [Microsoft AD DS 용어](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/Appendix-A--Reviewing-Key-AD-DS-Terms)

이 구분을 놓치면 “별도 OU니까 관리자 침해의 영향을 받지 않는다” 같은 잘못된 판단으로 이어진다.

## 구조를 검토할 때 남길 내용

예제 환경은 단일 Forest·단일 Domain에 `Users`, `Workstations`, `Servers` OU를 두는 것으로 설명한다. 이것이 모든 조직의 정답이라는 의미는 아니다.

설계서에는 각 OU의 목적, 관리 그룹, 위임 작업, 연결할 GPO를 기록한다. 독립적인 보안 책임이나 별도 Forest가 필요한 요구가 있다면 그 이유와 운영 비용도 따로 검토한다.

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.
