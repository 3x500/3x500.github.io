---
layout: single
title: "AD 19. gMSA 전환: KDS·호스트 허용·서비스 권한·원복 검증"
categories: [Active Directory]
tags: [AD, gMSA, Service Account, KDS]
toc: true
published: true
---

Windows 서비스가 도메인 리소스에 접근하려면 서비스 실행 계정의 권한이 필요할 수 있다. 이때 일반 사용자 계정을 오래 사용하면 암호 변경 시 서비스 설정도 함께 바꿔야 하고, 담당자가 바뀌면 사용처를 놓치기 쉽다.

서비스 계정은 사람의 업무 계정과 목적·수명주기를 구분해 관리한다. 비밀번호를 직접 입력하는 계정만 선택 가능한 것은 아니다. 가상 계정, sMSA, gMSA 등 서비스와 환경이 지원하는 방식을 검토한다. [Microsoft 서비스 계정](https://learn.microsoft.com/en-us/windows/security/identity-protection/access-control/service-accounts)

## gMSA가 해결하는 부분

Group Managed Service Account(gMSA)는 도메인이 암호를 관리하고 허용된 호스트가 이를 받아 서비스에 사용할 수 있는 계정이다. 여러 서버에서 같은 서비스 신원을 쓰는 구성에도 활용할 수 있다. 사람이 암호를 복사해 배포하는 부담을 줄인다. [Microsoft gMSA 개요](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-managed-service-accounts/group-managed-service-accounts/group-managed-service-accounts-overview)

암호 관리가 자동화돼도 과도한 리소스 권한이나 잘못된 SPN이 자동으로 해결되지는 않는다.

## 도입 전 확인할 조건

| 항목 | 확인 이유 |
|---|---|
| DC·기능 수준·호스트 OS | 지원 조건과 인증 기능 확인 |
| KDS 루트 키 | gMSA 암호 생성 기반 준비 |
| 사용 가능한 호스트 | 암호를 가져갈 수 있는 주체 제한 |
| 애플리케이션 지원 | 서비스가 gMSA 실행을 지원하는지 확인 |
| SPN·DNS·시간 | Kerberos와 대상 서비스 동작 확인 |

Microsoft 관리 안내는 도메인·Forest 기능 수준 2012 이상과 KDS 루트 키 등을 전제 조건으로 설명한다. 실제 서비스의 제품 버전·지원 조건도 따로 확인한다. 클러스터 서비스 자체와 그 위에서 실행되는 서비스의 지원은 구분해야 한다. [Microsoft gMSA 관리](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-managed-service-accounts/group-managed-service-accounts/manage-group-managed-service-accounts)

## 원본 도입 흐름의 완료 조건

```text
KDS 준비·복제 → gMSA 생성 → 암호 조회 허용 호스트 지정
             → 대상 호스트 설치·테스트 → 서비스 신원 전환 → 업무 검증
```

원본 보안 문서의 도입 순서를 단계별 완료 조건으로 재구성했다. 서비스 암호 관리와 서비스의 리소스 접근 권한을 별도 승인한다.

| 단계 | 확인 | 다음 단계 조건 |
|---|---|---|
| KDS | 기존 루트 키·유효 시점·DC 복제 | 각 DC에서 필요한 키가 준비됨 |
| 계정 | 이름·DNSHostName·SPN·지원 조건 | 앱이 지원하는 신원과 이름 일치 |
| 호스트 허용 | `PrincipalsAllowedToRetrieveManagedPassword`와 허용 그룹 | 지정 호스트만 암호를 받을 수 있음 |
| 호스트 설치 | AD 모듈·계정 준비·인증·서비스 로그온 | Test-ADServiceAccount와 호스트 정책 확인 |
| 서비스 전환 | 제품별 gMSA 표기와 구성·권한 | 시작·업무·재시작·계정 회전 이후 동작 |

KDS 키를 만들었다고 즉시 모든 DC에서 사용할 수 있는 것은 아니다. 공식 안내의 최대 10시간 대기와 복제 수렴 조건을 검토한다. 단일 DC 실습용 과거 유효시각 지정 예제를 운영 지름길로 적용하거나 기존 키를 삭제·재생성하지 않는다. [KDS 루트 키 준비 조건](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-managed-service-accounts/group-managed-service-accounts/create-the-key-distribution-services-kds-root-key)

암호 조회 허용 호스트는 민감한 경계다. 서버 관리 권한을 가진 주체가 서비스 신원과 키에 미칠 영향도 고려한다. gMSA를 여러 서버가 공유할 수 있다는 기능을 모든 서버의 공용 고권한 계정으로 사용하지 않는다.

## 호스트에서 준비 상태 확인

미리 gMSA 사용을 허용하고 설치한 대상 서버에서, Active Directory 모듈과 관리자 권한으로 실행하는 예제다.

```powershell
Test-ADServiceAccount -Identity gmsa-web
```

`True`는 해당 호스트에서 계정 사용 준비 상태를 확인한 결과다. 웹사이트나 배치가 끝까지 정상 처리됐다는 의미는 아니다. 이 글은 실행 결과를 첨부하지 않았다. [Test-ADServiceAccount](https://learn.microsoft.com/en-us/powershell/module/activedirectory/test-adserviceaccount)

## 전환 검증은 업무 동작까지 본다

기존 실행 계정·권한·SPN·서비스 설정과 원복 방법을 기록한다. 파일럿에서 계정 준비 상태를 확인한 뒤 서비스 시작, 리소스 접근, 재시작 후 동작까지 확인한다. 불필요한 호스트가 암호를 가져갈 수 없는지도 검증한다.

문제가 나면 원래 계정에 관리자 권한을 추가하는 식으로 해결하지 않는다. 계정 준비·서비스 로그온·업무 리소스 권한을 나누어 점검한다.

| 증상 | 다음 확인 |
|---|---|
| 계정 테스트 실패 | KDS·복제·호스트 허용 그룹·최근 토큰·시간·DNS |
| 테스트 성공, 서비스 시작 실패 | 제품 지원·계정 표기·로그온 권한·구성 파일·이벤트 |
| 서비스 시작, 파일/DB 접근 거부 | 리소스 ACL·DB 권한·실제 사용 계정 |
| Kerberos만 실패 | 클라이언트 접근 이름·SPN 보유 객체·키·암호화 호환성 |
| 일부 노드만 실패 | 호스트 허용·정책·설치·선택 DC·앱 버전 차이 |

전환 전 기존 신원·권한·SPN·복구 자격 증명 보관·제품 설정과 중단 시간을 기록한다. 원복할 때는 서비스 신원뿐 아니라 이번에 이동한 SPN·리소스 권한·노드 설정도 복구한다. 기존 계정은 파일럿 검증과 의존성 조사 전에 삭제하지 않고, 성공 후 불필요한 권한·자격 증명 사용처를 승인된 절차로 회수한다.

---

적용 기준: Windows Server의 gMSA. 도메인·Forest 기능 수준과 서비스 지원 조건은 본문 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.

[연재 목차]({% post_url 2026-08-13-ad-series-guide %})
