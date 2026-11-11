---
layout: single
title: "AD 07. Kerberos 심화: 티켓·SPN·시간·감사 로그로 실패 단계 찾기"
categories: [Active Directory]
tags: [AD, Kerberos, TGT, SPN]
toc: true
published: true
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

## AS·TGS·서비스 접근을 관찰 위치로 나눈다

```text
Client ── AS 교환 ── KDC/DC       : 사용자 인증과 TGT
Client ── TGS 교환 ─ KDC/DC       : TGT를 이용한 특정 SPN의 서비스 티켓
Client ── 서비스 티켓 ─ Service   : 대상 서비스 인증 후 리소스 권한 검사
```

암호와 티켓, 티켓을 보호하는 키는 서로 다르다. 암호가 매 요청마다 서비스에 그대로 전달되는 것으로 설명하지 않는다. 티켓을 발급받은 클라이언트와 티켓을 사용하는 서비스의 로그를 나눠 본다. [Kerberos의 세 주체와 동작](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview#how-kerberos-authentication-works)

| 관찰 위치 | 남길 정보 | 판정 범위 |
|---|---|---|
| 사용한 클라이언트 세션 | 계정·로그온 세션·접근 이름·현재 티켓·시각 | 그 세션의 캐시. 다른 서비스 계정 세션을 대신하지 않음 |
| 요청을 처리한 DC | 감사 정책, TGT·서비스 티켓 요청과 실패 코드 | 해당 KDC가 관찰한 요청. 모든 서비스 사용을 나타내지 않음 |
| 서비스 서버 | 서비스 로그·로그온 결과·실제 ACL | 인증과 리소스 허용/거부. 발급 성공만으로 정상 판정하지 않음 |

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

## 원본 보안 문서의 공격 표를 방어 점검으로 연결한다

| 원본의 위협 구분 | 운영에서 확인할 것 | 단정하지 않을 것 |
|---|---|---|
| AS-REP Roasting | 사전 인증을 요구하지 않는 계정의 존재와 사유 | 설정 또는 요청 한 건만으로 침해 확정 |
| Kerberoasting | SPN 보유 계정·서비스 암호 수명주기·지원 암호화·요청 패턴 | 서비스 티켓 요청 자체를 공격으로 판정 |
| Golden Ticket | KDC·고권한 계정·krbtgt 키 보호와 사고 복구 계획 | 일반 로그인 실패를 티켓 위조로 확정 |
| Silver Ticket | 서비스 계정 키·서비스 로그와 권한 | DC의 티켓 요청 로그가 없다는 이유만으로 안전 판정 |

방어 우선순위는 서비스 신원·권한·키 노출 경로와 정상 기준선에 따라 정한다. 암호화 정책 강제, 서비스 계정 전환, krbtgt 초기화는 호환성·복제·티켓 수명에 영향을 주므로 이 글의 즉시 실행 조치로 제시하지 않는다. 원본에 있는 간격·횟수만 가져와 사고 복구 절차를 대신하지 않는다.

4768은 TGT 관련 요청, 4769는 서비스 티켓 요청을 관찰하는 DC 감사 이벤트다. 정책과 버전·업데이트에 따른 필드 차이를 확인하고 계정·서비스·클라이언트 주소·상태·암호화 유형을 상관분석한다. RC4 관찰은 개선·조사 대상이며 그 한 값만으로 공격 성공을 증명하지 않는다. [이벤트 4768](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4768), [이벤트 4769](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4769)

## SPN 변경 전에 읽을 것

AD 관리 도구가 설치된 관리 PC의 관리자 명령 프롬프트에서, 디렉터리 조회 권한으로 실행할 예제다. `HTTP/app01.ad.example.com`은 실제 앱이 사용하는 서비스 이름으로 바꾼다.

```text
setspn -Q HTTP/app01.ad.example.com
```

서비스 실행 계정과 SPN 보유 객체가 일치하는지 확인한다. 결과가 없거나 둘 이상이면 접근 이름·자동 등록·다중 도메인 범위까지 검토한다. `-Q` 조회와 `-S` 등록은 다른 작업이다. 잘못된 객체에 SPN을 추가하면 다른 서비스에 영향을 줄 수 있다. [setspn의 조회·등록·중복 확인](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn)

| 증상 | 다음 확인 | 중단·원복 기준 |
|---|---|---|
| TGT 없음·발급 실패 | 세션·온라인 인증·KDC 도달성·계정·시간·실패 코드 | 원인 미확정 상태에서 암호·정책 일괄 변경 보류 |
| 특정 서비스 티켓 실패 | 접근 이름·SPN·계정·지원 암호화 | 변경한 SPN/정책이 다른 서비스에 영향 주면 기록한 설정으로 복원 |
| 서비스 티켓은 있으나 거부 | 서비스가 티켓을 읽을 키·로그·ACL | 인증 거부와 파일 권한 거부를 분리 |
| 간헐 실패 | 사용 DC·서비스 노드·시각·캐시·키 변경 시점 | 키·계정 변경의 영향이 수렴하기 전 추가 변경 중단 |

원본의 “2분” 같은 경보 예제와 Kerberos의 허용 시간 정책은 구분한다. 기본 5분이라는 문서값도 현재 적용값·시간 원본을 조회하는 작업을 대신하지 않는다. 티켓 삭제·재인증은 세션에 영향을 주므로 운영 전체에 `klist purge`를 일괄 수행하지 않는다.

## 인증과 파일 권한을 분리해 진단한다

티켓 요청 실패라면 DNS·시간·SPN·계정 상태를 살핀다. 서비스가 사용자를 인증했는데 파일 접근만 거부된다면 그룹과 공유·파일 권한도 확인한다. 사용한 로그온 세션과 접근 이름을 기록해야 두 문제를 섞지 않을 수 있다.

---

적용 기준: Windows Server 2019 이상 AD DS의 개념·운영. 기능별 지원 조건은 본문과 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.

[연재 목차]({% post_url 2026-08-13-ad-series-guide %})
