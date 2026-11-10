---
layout: single
title: "AD 14. DC 복제와 데이터 일관성"
categories: [Active Directory]
tags: [AD, Replication, repadmin, SYSVOL]
toc: true
published: false
---

DC를 두 대 운영하는데 사용자 변경 내용이 다르게 보이면 어느 서버의 정보가 맞는지부터 헷갈릴 수 있다. AD는 여러 DC의 데이터를 복제해 유지한다. 변경이 모든 DC에 즉시 나타나는 것은 아니다.

각 지점이 같은 명부의 사본을 관리하고 변경 사항을 전달하는 모습에 가깝다. 다만 복제는 파일 전체를 단순 복사하는 작업이 아니며, 디렉터리의 변경 정보와 복제 토폴로지를 사용한다. [Microsoft 복제 개념](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/replication/active-directory-replication-concepts)

## 무엇을 비교하는지 먼저 정한다

| 대상 | 구분할 내용 |
|---|---|
| 도메인 디렉터리 데이터 | 해당 도메인 계정·그룹 등의 변경 |
| 스키마·구성 | Forest 공통 구조·구성 정보 |
| DNS 애플리케이션 파티션 | 설정된 복제 범위의 DNS 데이터 |
| SYSVOL | 정책 파일·스크립트 등의 파일 복제 |

GPO의 AD 정보와 SYSVOL 파일은 복제 경로가 구분된다. `repadmin` 결과가 정상이라고 SYSVOL도 정상이라고 단정하지 않는다. 반대로 공유 폴더가 보인다고 모든 AD 복제가 정상인 것도 아니다. [Microsoft GPO 복제 설명](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-policy/group-policy-processing)

## 요약에서 상세로 내려간다

AD 관리 도구가 설치된 관리 PC 또는 DC에서 필요한 조회 권한으로 실행하는 예제다. 실제 결과는 포함하지 않았다.

```text
repadmin /replsummary
repadmin /showrepl DC01
```

요약에서는 실패가 있는 DC와 지연 범위를 찾고, 상세에서는 복제 파트너·명명 컨텍스트·최근 성공 시각·오류를 본다. 대상 DC 한 대만 확인한 결과를 Forest 전체 상태로 일반화하지 않는다. [Microsoft 복제 문제 해결](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/troubleshoot/troubleshooting-active-directory-replication-problems)

## 실패 원인을 나누어 확인한다

이름 확인, 네트워크 통신, 인증, 데이터베이스·서비스, 토폴로지 문제는 서로 다른 원인이다. 오류 코드와 실패 구간을 기록하고 관련 DC 양쪽에서 같은 시간대의 상태를 비교한다.

오랫동안 성공하지 못한 파트너가 있다면 강제 동기화부터 반복하지 않는다. 단순 연결 장애인지, 오래된 DC·객체 문제가 있는지 확인해야 한다. 장기 장애에 적용할 조치는 복제 중단 기간과 디렉터리 상태에 따라 달라진다.

변경을 검증할 때는 같은 테스트 객체를 각 DC에 명시적으로 조회한다. 클라이언트가 다른 DC를 선택하거나 캐시를 보는 상황과 실제 데이터 복제 지연을 구분할 수 있다.

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.
