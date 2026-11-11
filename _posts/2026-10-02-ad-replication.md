---
layout: single
title: "AD 14. 복제 장애 진단: 파티션·파트너·성공 시각과 장기 중단 판단"
categories: [Active Directory]
tags: [AD, Replication, repadmin, SYSVOL]
toc: false
toc_sticky: true
published: true
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

## 파트너와 데이터 범위를 연결한다

```text
변경을 받은 DC → 해당 파티션의 복제 토폴로지 → 대상 DC
                    ├ Domain / Configuration / Schema
                    └ 참여하는 DNS 등 Application Partition
GPO 파일           → SYSVOL의 별도 파일 복제
```

KCC는 Site·연결 상태·비용·일정을 바탕으로 복제 연결을 구성한다. 모든 DC가 서로 직접 연결돼야 정상이라는 뜻은 아니다. 자동 연결을 수동으로 수정하면 KCC 관리 동작이 달라질 수 있으므로, 연결을 추가하기 전에 토폴로지와 실패 원인을 확인한다. [연결 객체와 KCC](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/replication/active-directory-replication-concepts#connection-object)

| 조회에서 볼 항목 | 질문 | 다음 행동 |
|---|---|---|
| Destination·Source | 어느 DC가 어느 파트너에서 받지 못했는가? | 양쪽 이름·서비스·연결 경로 비교 |
| Naming Context | 같은 파티션을 비교하는가? | Domain과 DNS 파티션 문제를 분리 |
| Last Success·Last Attempt | 성공 이후 언제부터 실패했는가? | 변경·회선·재시작 시각과 연결 |
| Error·연속 실패 | 어떤 코드가 반복되는가? | DNS·RPC·권한·시간·DB 등 해당 원인 조사 |
| 관찰 범위 | 어느 DC·기간·파트너를 검사했는가? | 누락 DC와 의도한 점검 제외를 명시 |

## 실패 원인을 나누어 확인한다

이름 확인, 네트워크 통신, 인증, 데이터베이스·서비스, 토폴로지 문제는 서로 다른 원인이다. 오류 코드와 실패 구간을 기록하고 관련 DC 양쪽에서 같은 시간대의 상태를 비교한다.

오랫동안 성공하지 못한 파트너가 있다면 강제 동기화부터 반복하지 않는다. 단순 연결 장애인지, 오래된 DC·객체 문제가 있는지 확인해야 한다. 장기 장애에 적용할 조치는 복제 중단 기간과 디렉터리 상태에 따라 달라진다.

변경을 검증할 때는 같은 테스트 객체를 각 DC에 명시적으로 조회한다. 클라이언트가 다른 DC를 선택하거나 캐시를 보는 상황과 실제 데이터 복제 지연을 구분할 수 있다.

## 장기 중단은 즉시 강제 복제로 처리하지 않는다

원본 구축 가이드의 “복제 지연 24시간”은 운영 경보 예제가 될 수 있지만 Tombstone Lifetime과 같지 않다. 해당 Forest의 `tombstoneLifetime`, 마지막 성공 복제, 현재 시각과 실제 오류를 함께 본다. 과거 생성·업그레이드 조건에 따라 달라질 수 있어 모든 환경을 180일이라고 고정하지 않는다. 8614는 마지막 복제 이후 기간이 수명 조건을 넘은 문제를 조사하는 단서다. [복제 오류 8614와 실제 수명 확인](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/replication-error-8614)

AD 객체 삭제 보존과 DFSR Content Freshness는 서로 다른 보호 기능이다. SYSVOL의 복제 중단은 DFS Replication 로그·복제 폴더 상태·적용 설정으로 판단한다. `repadmin`만으로 파일 복제 보호를 해제하거나 정상 판정하지 않는다. [SYSVOL·DFSR 상태와 초기화 주의사항](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/troubleshoot-missing-sysvol-and-netlogon-shares)

| 상황 | 먼저 보존·확인 | 중단·복구 판단 |
|---|---|---|
| 단기 통신 실패 | 원인 코드·변경·양쪽 로그·마지막 성공 | 경로 복구 후 수렴 관찰. 원인 없이 강제 동기화 반복하지 않음 |
| 오래된 DC 재연결 | 수명·시간·삭제 이력·보호 이벤트 | 확산 전 지원 절차로 정리·재배포 필요성 판단 |
| Lingering Object 의심 | 파티션·정상 기준 DC·조사 범위 | 기준이 확정되기 전 제거 명령 실행 보류 |
| SYSVOL 누락 | 공유·DFSR 이벤트·각 DC 상태·백업 | AD 복제 정상 여부와 별도로 복구 계획 수립 |

원본의 장애 대응 절차를 옮길 때는 “기준 DC”를 먼저 확정한다. 잘못된 기준으로 복구·제거하면 정상 데이터에 영향을 줄 수 있다. 이 글은 강제 동기화·Lingering Object 삭제·DFSR 권위 복구의 실행 Runbook이 아니다. 복구 후에는 같은 객체·파티션의 수렴과 SYSVOL, 인증·GPO·DNS를 각각 재검증한다.

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.

[연재 목차]({% post_url 2026-08-13-ad-series-guide %})
