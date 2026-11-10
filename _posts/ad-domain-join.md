---
layout: single
title: "AD 09. 도메인 가입과 컴퓨터 계정"
categories: [Active Directory]
tags: [AD, Domain Join, Secure Channel]
toc: true
published: false
---

PC를 도메인에 가입시키는 일은 로그인 화면에 회사 계정을 입력할 수 있게 하는 것 이상이다. AD의 컴퓨터 계정과 장비의 관계를 만들고 도메인 관리에 참여시키는 과정이다.

사용자 계정이 사람의 신원이라면 컴퓨터 계정은 장비의 신원에 해당한다. 사용자 비밀번호를 재설정하는 것과 PC의 도메인 신뢰 문제를 해결하는 일이 다른 이유다. [Microsoft 도메인 가입 안내](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/join-computer-to-domain)

## 가입 전에 확인할 조건

1. 클라이언트 OS 에디션이 도메인 가입을 지원하는지 확인한다.
2. 내부 DNS로 도메인과 DC를 찾을 수 있는지 확인한다.
3. DC까지 필요한 통신과 시간 동기화 상태를 확인한다.
4. 가입 계정에 대상 위치의 컴퓨터 계정 생성·재사용 권한이 있는지 확인한다.

모든 가입 작업에 Domain Admin 계정을 사용하는 대신 업무에 필요한 권한을 위임하는 방향을 검토한다. 기존 컴퓨터 계정을 재사용할 때는 도메인 가입 강화 업데이트의 계정 소유권 조건도 영향을 줄 수 있다. [Microsoft 도메인 가입 문제 해결](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/active-directory-domain-join-troubleshooting-guidance)

## 실패 기록은 클라이언트에서도 찾는다

가입 실패 시 오류 문구와 시각을 먼저 기록한다. 클라이언트의 `%windir%\debug\NetSetup.log`에는 가입 과정의 정보가 남는다. DC 검색 실패인지, 계정 재사용·권한 문제인지 구분하는 데 활용한다. [Microsoft 가입 인증 오류](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/domain-join-authentication-errors)

오류가 났다고 바로 컴퓨터 계정을 삭제하고 다시 가입하지 않는다. 기존 객체에 연결된 권한과 관리 정보, 장비 재시작 영향까지 확인해야 한다.

## 가입 후 보안 채널 확인

아래는 도메인에 가입한 멤버 PC에서 관리자 권한의 Windows PowerShell로 실행할 진단 예제다. DC에서 사용하는 명령으로 일반화하지 않는다.

```powershell
Test-ComputerSecureChannel -Verbose
```

`True`는 해당 시점에 보안 채널 확인이 성공했다는 의미다. DNS·정책·업무 애플리케이션 전체가 정상이라는 뜻은 아니다. `False`라면 연결·계정 상태를 확인한 뒤 복구 방식을 결정한다. 이 글에서는 상태를 바꾸는 `-Repair` 실행을 포함하지 않는다. [Test-ComputerSecureChannel](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/test-computersecurechannel?view=powershell-5.1)

가입 후에는 올바른 OU 배치, 도메인 사용자 로그인, 필요한 GPO와 리소스 접근까지 확인한다. 가입 성공 창만으로 업무 전환을 끝내지 않는다.

---

적용 기준: AD DS와 도메인 가입을 지원하는 Windows 클라이언트·멤버 서버. 보안 채널 예제는 Windows PowerShell 5.1.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.
