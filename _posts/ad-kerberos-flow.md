---
layout: single
title: "AD 07. Kerberos 로그인과 티켓의 흐름"
categories: [Active Directory]
tags: [AD, Kerberos, TGT, SPN]
toc: true
published: false
---

도메인에 로그인한 뒤 파일 서버를 열 때마다 비밀번호를 다시 입력하지 않는 이유는 무엇일까. Kerberos에서는 인증을 거쳐 발급받은 티켓을 서비스 접근에 사용한다.

입장권 교환권과 개별 시설 이용권을 떠올리면 쉽다. Ticket Granting Ticket(TGT)은 서비스 티켓을 요청하는 데 쓰이고, 서비스 티켓은 특정 서비스에 제시한다. 티켓을 가졌다는 사실과 그 서비스의 모든 자료를 읽을 권한이 있다는 것은 별개다. [Microsoft Kerberos 개요](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview)

## 세 주체로 나눠 본다

| 주체 | 역할 |
|---|---|
| 클라이언트 | 사용자를 대신해 인증·서비스 티켓 요청 |
| KDC | 계정 확인과 티켓 발급, 도메인에서는 DC에서 동작 |
| 서비스 | 제시된 티켓으로 신원을 확인하고 리소스 권한 판단 |

사용자 인증과 TGT 발급, 서비스 티켓 요청, 대상 서비스 접근으로 나누어 보면 어느 구간에서 실패했는지 찾기 쉽다. Windows의 실제 동작에는 캐시·위임·정책 등 추가 조건도 있다.

## 현재 세션의 티켓 확인

도메인 사용자로 로그인한 PC의 명령 프롬프트에서 현재 세션을 확인하는 예제다. 다른 계정의 세션과 고급 옵션은 추가 권한이 필요하다.

```text
klist
klist tgt
```

티켓의 서버 이름, 시작·만료 시각, 암호화 유형을 확인한다. 특정 서비스 티켓이 없다고 바로 장애라고 판단하지 않는다. 아직 그 서비스에 접근하지 않았거나 다른 인증 방식이 사용됐을 수 있다. 이 글은 실제 티켓 출력을 공개하지 않는다. [Microsoft klist](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/klist)

## SPN과 시간도 인증 조건이다

Service Principal Name(SPN)은 서비스를 식별하는 이름이다. 사용한 이름과 서비스 계정의 SPN 관계가 맞지 않거나 중복되면 티켓 발급·사용 과정에 문제가 생길 수 있다. 조회와 등록은 구분하고, 원인을 확인하기 전에 SPN을 추가하지 않는다. [Microsoft setspn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn)

시간 차이도 확인한다. Kerberos의 시간 허용 범위와 티켓 유효성에 영향을 주므로, 로그인 실패 시 계정 상태만 보지 말고 DC·클라이언트의 시간 원본과 오차를 확인한다. [Microsoft Kerberos 정책](https://learn.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/maximum-tolerance-for-computer-clock-synchronization)

## 인증과 파일 권한을 분리해 진단한다

티켓 요청 실패라면 DNS·시간·SPN·계정 상태를 살핀다. 서비스가 사용자를 인증했는데 파일 접근만 거부된다면 그룹과 공유·파일 권한도 확인한다. 사용한 로그온 세션과 접근 이름을 기록해야 두 문제를 섞지 않을 수 있다.

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.
