---
layout: single
title: "AD 02. Forest·Domain·OU 설계: 위임과 보안 경계의 선택 기준"
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

## 원본 토폴로지 선택표를 요구와 비용으로 바꾼다

![원본 구축 문서의 토폴로지 선택 후보 도식]({{ '/images/ad-deep-dive/a-002.png' | relative_url }})

원본 도식은 단일 운영, 지역 위임, 독립 보안 경계를 토폴로지 후보에 연결한다. 해외 법인이 있다는 사실만으로 Multi Domain이 필수인 것은 아니다. 또 “Forest Trust”는 독립 Forest 사이의 인증 신뢰를 설정하는 선택이다. 신뢰를 만들면 그 방향과 범위에 따라 접근 경로가 생기므로 격리 요구를 다시 평가한다.

| 요구 | 먼저 검토할 선택 | 추가 비용·판정 |
|---|---|---|
| 부서별 계정 관리·정책 | 같은 Domain의 OU와 그룹 위임 | 필요한 객체·작업만 위임했는지 검증 |
| 지역별 운영·데이터 복제 범위 | OU·Site로 충족 가능한지 평가 후 추가 Domain 검토 | 도메인별 DC·DNS·정책·백업 운영 증가 |
| 독립 서비스 관리자·스키마·보안 책임 | 별도 Forest | 계정·권한·이관·복구 체계 분리 비용 |
| 독립 Forest 자원 공유 | 필요한 Trust와 리소스 ACL | 방향·선택적 인증·이름 해석·권한을 별도 검증 |

같은 Forest는 스키마와 구성을 공유한다. 도메인 추가로 지역 운영 책임을 나눠도 Forest 수준의 영향에서 독립되는 것은 아니다. 별도 Forest도 관리 계정·하이퍼바이저·백업을 공유하면 현실의 침해 경로가 남는다. [Forest의 보안 경계](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/appendix-a--reviewing-key-ad-ds-terms#active-directory-forest)

원본의 전사 Baseline은 **각 도메인에 배포·연결하고 동일성을 관리할 표준**으로 해석한다. 부모 도메인의 GPO가 자식 도메인으로 자동 상속되지는 않는다. [GPO의 도메인 경계](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/policy/linking-gpos-to-active-directory-containers)

Trust 검증 성공과 SID Filtering·Selective Authentication 설정 확인도 별도다. 신뢰 유형·방향·속성을 읽고 승인된 사용자로 의도한 리소스의 허용·거부를 시험한다. 신뢰 검증 명령 한 번으로 보안 옵션 전체를 확인했다고 쓰지 않는다. [netdom trust 옵션](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/netdom-trust), [Get-ADTrust](https://learn.microsoft.com/en-us/powershell/module/activedirectory/get-adtrust)

## 논리 구조를 실제 조회와 연결한다

AD 관리 모듈이 있는 관리 PC에서 해당 Forest·Domain을 읽을 수 있는 계정으로 실행하는 예제다. 서버·도메인 값은 가상이며 기능 수준 변경은 수행하지 않는다.

```powershell
Get-ADForest -Server DC01 |
    Select-Object RootDomain, Domains, ForestMode
Get-ADDomain -Server DC01 |
    Select-Object DNSRoot, ParentDomain, DistinguishedName, DomainMode
```

Forest의 도메인 목록과 조회한 Domain을 대조한다. 결과에 도메인이 둘 이상이라는 사실은 물리 지점이 둘이라는 뜻이 아니다. 조회된 기능 수준 역시 OS 버전이나 모든 신기능 사용 여부를 대신하지 않는다. [Get-ADForest](https://learn.microsoft.com/en-us/powershell/module/activedirectory/get-adforest), [Get-ADDomain](https://learn.microsoft.com/en-us/powershell/module/activedirectory/get-addomain)

| 검토 단계 | 확인 | 중단·재설계 기준 |
|---|---|---|
| 논리 설계 | 계정·정책·위임 책임을 Domain/OU에 연결 | 부서명만 있고 위임 작업·책임자가 없음 |
| 권한 검증 | 테스트 위임자가 필요한 작업만 성공 | 대상 밖 수정·상위 권한 획득 가능 |
| 정책 검증 | 도메인별 링크와 파일럿 결과 | 부모 Domain에서 자동 전파된다고 가정 |
| 구조 변경 | DN 의존 앱·GPO·ACL·그룹·백업 영향 | 기존 OU 이동·삭제 영향과 원복 경로 미확인 |

OU 이동은 비교적 작은 구조 변경이어도 정책과 ACL 상속, DN 기반 검색에 영향을 준다. 이동 전 객체 위치·연결 정책·권한을 남기고 테스트 대상으로 적용한다. Domain/Forest 통합·분리는 단순 OU 이동 원복 절차로 처리하지 않는다.

## 구조를 검토할 때 남길 내용

예제 환경은 단일 Forest·단일 Domain에 `Users`, `Workstations`, `Servers` OU를 두는 것으로 설명한다. 이것이 모든 조직의 정답이라는 의미는 아니다.

설계서에는 각 OU의 목적, 관리 그룹, 위임 작업, 연결할 GPO를 기록한다. 독립적인 보안 책임이나 별도 Forest가 필요한 요구가 있다면 그 이유와 운영 비용도 따로 검토한다.

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.

[연재 목차]({% post_url 2026-11-10-ad-series-guide %})
