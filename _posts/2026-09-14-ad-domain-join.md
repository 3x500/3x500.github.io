---
layout: single
title: "AD 09. 도메인 가입 심화: 객체 재사용·보안 채널·업무 전환 검증"
categories: [Active Directory]
tags: [AD, Domain Join, Secure Channel]
toc: true
published: true
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

## 컴퓨터 계정 생성과 재사용을 구분한다

원본 구축 가이드의 DNS·통신·시간·승인 계정 체크리스트를 가입 단계에 맞추면 다음과 같다. 원본의 확인 기준은 실제 대상 OS·업데이트·위임 정책에 맞게 검토한다.

| 단계 | 결정·확인 | 증거 |
|---|---|---|
| 장비 준비 | 지원 OS, 장비명·중복 여부, 로컬 복구 접근 | 대상 장비 식별과 준비 기록 |
| 이름·통신 | 내부 DNS, DC 검색, 필요한 프로토콜 | SRV·후보 주소·구간별 연결·실패 코드 |
| 객체 준비 | 새 생성인지 사전 생성 객체 재사용인지 | 대상 OU·객체 소유자·가입 위임자 |
| 가입·재시작 | 승인 계정과 전환 시점 | NetSetup 시각·오류, 재시작 후 도메인 상태 |
| 업무 인수 | 정책·접근·관리 도구·백업 에이전트 등 | 대표 업무와 보안 채널 테스트 |

재사용은 같은 이름의 객체가 있다는 이유만으로 허용되지 않는다. Microsoft의 도메인 가입 강화에 따라 기존 객체 소유권과 신뢰할 소유자 정책, 클라이언트·DC 업데이트가 영향을 준다. 과거 우회 레지스트리 값을 최신 해결책으로 가져오지 않는다. 소유자·정책을 고치는 것과 컴퓨터 계정을 삭제하는 것은 영향이 다르다. [KB5020276 도메인 가입 강화](https://support.microsoft.com/en-us/servicing/os/windows/2022/10/kb5020276-netjoin-domain-join-hardening-changes)

## NetSetup의 실패 시점을 읽는다

클라이언트의 관리자 권한으로 로그를 읽고 실패 시각·대상 도메인·선택 DC·작업·오류를 연결한다. 로그에는 이름·경로·주소가 들어가므로 외부 공유 전 가린다. 원인 후보를 아래처럼 나누고 앞 단계가 확인되지 않았으면 뒤 단계의 계정을 초기화하지 않는다.

| 실패 구간 | 다음 확인 |
|---|---|
| DC를 찾지 못함 | 실제 DNS 서버·SRV·후보 주소·DC Locator |
| DC 도달·RPC 단계 | 구간별 포트·방화벽·서비스·로그의 오류 코드 |
| 계정 생성·재사용 거부 | 위임 권한·기존 객체·소유자·강화 정책 |
| 가입 뒤 신뢰 오류 | 멤버 장비와 객체 상태·네트워크·보안 채널 |
| 가입 성공, 업무 실패 | OU·GPO·사용자 토큰·리소스·앱의 별도 검증 |

도메인 가입은 DNS·Kerberos·LDAP·SMB·RPC 등 여러 통신에 의존한다. Ping 또는 TCP 포트 하나의 성공을 전체 가입 가능 판정으로 쓰지 않는다. [가입 진단 체크리스트와 프로토콜](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/active-directory-domain-join-troubleshooting-guidance)

## 가입 후 보안 채널 확인

아래는 도메인에 가입한 멤버 PC에서 관리자 권한의 Windows PowerShell로 실행할 진단 예제다. DC에서 사용하는 명령으로 일반화하지 않는다.

```powershell
Test-ComputerSecureChannel -Verbose
```

`True`는 해당 시점에 보안 채널 확인이 성공했다는 의미다. DNS·정책·업무 애플리케이션 전체가 정상이라는 뜻은 아니다. `False`라면 연결·계정 상태를 확인한 뒤 복구 방식을 결정한다. 이 글에서는 상태를 바꾸는 `-Repair` 실행을 포함하지 않는다. [Test-ComputerSecureChannel](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/test-computersecurechannel?view=powershell-5.1)

가입 후에는 올바른 OU 배치, 도메인 사용자 로그인, 필요한 GPO와 리소스 접근까지 확인한다. 가입 성공 창만으로 업무 전환을 끝내지 않는다.

## 재가입을 원복 명령으로 일반화하지 않는다

전환 전 장비·객체 식별자, OU, 정책, 업무 계정·프로필·암호화·서비스 의존성, 재시작 시점과 로컬 복구 접근을 기록한다. 승인된 파일럿 장비에서 먼저 검증한다. 기존 객체 삭제·도메인 탈퇴·재가입은 SID·권한·관리 정보·사용자 업무에 영향을 줄 수 있다.

가입 중단이나 신뢰 오류가 생기면 로그와 객체 상태를 보존하고 단계에 맞는 지원 절차를 선택한다. 복구 후에는 보안 채널뿐 아니라 기존 사용자 프로필 접근, 정책, 파일·앱과 관리 에이전트를 재검증한다. 원본의 Go/No-Go 요구를 단순히 “가입 성공” 한 항목으로 축소하지 않는다.

---

적용 기준: AD DS와 도메인 가입을 지원하는 Windows 클라이언트·멤버 서버. 보안 채널 예제는 Windows PowerShell 5.1.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.

[연재 목차]({% post_url 2026-08-13-ad-series-guide %})
