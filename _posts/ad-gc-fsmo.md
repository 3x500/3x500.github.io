---
layout: single
title: "AD 16. GC와 FSMO 역할 이해하기"
categories: [Active Directory]
tags: [AD, Global Catalog, FSMO]
toc: true
published: false
---

“DC가 두 대니까 역할도 완전히 같다”는 설명에는 빠진 부분이 있다. 일반적인 디렉터리 변경을 여러 DC에서 처리하더라도 GC 설정과 FSMO 역할 보유 상태는 별도로 확인해야 한다.

GC는 여러 도메인의 정보를 찾는 통합 색인에 가깝다. FSMO는 충돌을 피하기 위해 특정 작업의 책임자를 정해 두는 방식이다. 둘은 서로 다른 기능이다.

## Global Catalog의 역할

GC는 자기 도메인 데이터와 다른 도메인 객체의 일부 속성을 보유해 Forest 범위 검색을 지원한다. 모든 도메인의 모든 속성을 완전히 복사해 둔 저장소로 설명하면 안 된다. 여러 도메인에서 로그온·검색 요구와 WAN 조건을 고려해 배치한다. [Microsoft GC 배치](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/planning-global-catalog-server-placement)

## FSMO 5개 역할

| 범위 | 역할 | 주된 책임 |
|---|---|---|
| Forest | Schema Master | 스키마 변경 |
| Forest | Domain Naming Master | 도메인·관련 파티션 추가·제거 |
| Domain | RID Master | RID 풀 할당 |
| Domain | PDC Emulator | 암호 변경 우선 복제·인증 지원, 시간 계층 등 |
| Domain | Infrastructure Master | 다른 도메인 객체 참조 관련 관리 |

역할 장애의 영향은 작업마다 다르다. 역할 하나가 중지됐다고 모든 인증이 즉시 중단되는 것은 아니며, 반대로 다른 DC가 있다는 이유로 무시할 수도 없다. [Microsoft FSMO 역할](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/fsmo-roles)

## 현재 역할 확인

Active Directory 모듈이 설치된 관리 PC에서 디렉터리 조회 권한으로 실행하는 예제다.

```powershell
Get-ADForest | Select-Object SchemaMaster, DomainNamingMaster
Get-ADDomain | Select-Object PDCEmulator, RIDMaster, InfrastructureMaster
Get-ADDomainController -Filter * |
    Select-Object HostName, Site, IsGlobalCatalog
```

첫 명령과 두 번째 명령은 기본 접속 환경의 Forest·Domain을 대상으로 한다. 여러 환경을 관리하면 대상 컨텍스트를 명시해 조회한다. [Get-ADDomain](https://learn.microsoft.com/en-us/powershell/module/activedirectory/get-addomain), [Get-ADDomainController](https://learn.microsoft.com/en-us/powershell/module/activedirectory/get-addomaincontroller)

## 이전과 강제 점유를 구분한다

기존 보유 DC가 정상이고 계획적으로 옮기는 상황에서는 Transfer를 검토한다. Seize는 기존 역할 보유자를 사용할 수 없는 상황에서 영향과 복귀 방침을 고려해 판단한다. 일시적인 DC 장애에 바로 강제 점유를 적용하는 것은 적절하지 않을 수 있다. [Microsoft 역할 이전·점유 판단](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/fsmo-placement-and-optimization-on-ad-dcs)

운영 문서에는 역할 위치와 대체 대상뿐 아니라 판단 책임자, 사전 점검, 처리 후 복제·역할별 기능 확인까지 남긴다.

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.
