---
layout: single
title: "AD 21. 일상 점검과 장애 진단 순서"
categories: [Active Directory]
tags: [AD, dcdiag, repadmin, DNS, Troubleshooting]
toc: true
published: false
---

사용자가 “AD가 안 된다”고 말할 때 실제 증상은 로그인 실패, 가입 실패, 파일 접근 거부, 정책 미적용 등으로 다르다. 모든 항목을 한 번에 수정하기보다 실패 범위를 먼저 좁힌다.

어느 사용자·장비·지점에서 언제부터 어떤 작업이 실패하는지 확인한다. 전체 장애인지 일부 대상의 문제인지에 따라 먼저 볼 항목이 달라진다.

## 증상에서 출발한다

| 증상 | 우선 확인할 것 |
|---|---|
| 도메인 가입 실패 | DNS·DC 검색, 통신, 가입 권한, NetSetup 로그 |
| 인증 실패 | 계정 상태, DNS·시간, 접속 DC와 인증 로그 |
| 파일 접근 거부 | 인증 성공 여부, 그룹·세션·공유·파일 권한 |
| GPO 미적용 | 대상 OU·링크·필터, gpresult, SYSVOL·복제 |
| DC별 결과 불일치 | 같은 객체의 조회 대상과 복제 상태 |

이는 출발점이지 모든 원인을 판정하는 표는 아니다. 특히 복제 장애는 DNS·네트워크·인증 등 여러 원인으로 발생할 수 있다. [Microsoft 복제 진단](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/troubleshoot/troubleshooting-active-directory-replication-problems)

## DC 기본 진단 예제

DC의 관리자 권한 명령 프롬프트에서 실행할 예제다. 원격·전체 범위 조회는 별도 권한과 통신 조건을 확인한다.

```text
dcdiag /test:Advertising /test:Services /v
dcdiag /test:DNS /DnsBasic /v
repadmin /replsummary
```

여기서는 DNS 기본 진단을 명시했다. `dcdiag`의 다른 DNS 검사에는 동적 업데이트 테스트처럼 테스트 레코드를 등록·삭제하는 동작도 있으므로 모든 옵션을 순수 조회로 취급하지 않는다. 출력은 실제로 수집하지 않았다. [Microsoft DCDiag 옵션](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/dcdiag)

## 시간 상태는 값과 원본을 함께 본다

문제가 있는 장비에서 로컬 관리자 권한으로 조회한다.

```text
w32tm /query /status
w32tm /query /source
```

동기화 원본과 마지막 동기화 등 상태를 확인한다. 현재 시각이 비슷해 보여도 원본이 의도한 계층과 다르면 이후 문제가 반복될 수 있다. 이 조회만으로 상대 장비와의 실제 오차까지 측정했다고 보지는 않는다. [Microsoft Windows Time 도구](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings)

## 정상 기준을 기록한다

실패 수, 최근 성공 시각, 조회한 DC, 기대한 응답을 남겨 전날·정상 장비와 비교한다. 이벤트 로그의 오류 한 줄만 지우는 방식보다 범위와 변화 추적이 중요하다.

서비스 재시작·강제 복제·계정 재설정은 상태를 바꾸는 조치다. 원인과 영향, 확인할 결과를 정한 뒤 수행하고, 정상 여부는 사용자 업무 동작까지 확인한다. 공개 글에는 원본 로그 대신 진단 순서와 판단 근거를 정리한다.

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.
