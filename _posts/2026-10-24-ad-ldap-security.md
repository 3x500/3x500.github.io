---
layout: single
title: "AD 20. LDAP 하드닝: 바인드 방식별 서명·TLS·CBT 적용과 호환성"
categories: [Active Directory]
tags: [AD, LDAP Signing, TLS, Channel Binding]
toc: true
published: true
---

“LDAP 보안을 켠다”는 표현만으로는 어떤 위협을 막는지 알기 어렵다. 서명, 암호화, 채널 바인딩은 적용 대상과 목적이 다르다.

서명을 봉인의 훼손 확인에, 암호화를 내용을 읽지 못하게 하는 보호에 비유할 수 있다. 채널 바인딩은 인증이 지금 연결된 TLS 세션과 맞는지 연결해 확인하는 장치다. 각각을 동일한 설정으로 보면 호환성 점검도 빠진다.

## 보호 계층을 구분한다

| 보호 | 주요 목적 | 적용 맥락 |
|---|---|---|
| LDAP 서명 | 메시지 무결성·인증성 | TLS를 사용하지 않는 SASL 세션 |
| SASL sealing | 기밀성 등 | 해당 보호를 협상한 SASL 세션 |
| TLS | 전송 기밀성·무결성·서버 인증 | LDAPS 또는 STARTTLS |
| Channel Binding | 인증과 TLS 연결의 결합 | TLS 위의 SASL 인증 |

서명만으로 내용을 숨길 수는 없다. Simple Bind의 자격 증명을 전송할 때는 TLS 보호가 중요하며, 채널 바인딩이 Simple Bind에 같은 방식으로 적용되는 것도 아니다. [Microsoft LDAP 보호 계층](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing), [Microsoft 채널 바인딩](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-channel-binding)

## 포트 번호로만 판단하지 않는다

일반적으로 LDAPS는 636, STARTTLS는 389에서 사용할 수 있다. SASL로 보호하는 연결도 있으므로 “389면 평문, 636이면 모든 보안 설정 완료”라고 단정하지 않는다. 인증서 신뢰·이름·유효기간과 애플리케이션의 검증 방식도 확인한다.

## 강제 적용 전에 연동 현황을 조사한다

| 조사 항목 | 남길 내용 |
|---|---|
| 애플리케이션 | 담당자와 사용 목적 |
| 접속 정보 | 대상 서버·포트·인증 방식 |
| 보호 방식 | TLS·서명·CBT 지원 여부 |
| 검증 결과 | 성공·실패 조건과 오류 |
| 변경 방안 | 클라이언트 수정·인증서 배포·예외 종료 시점 |

DC의 관련 이벤트와 애플리케이션 로그를 함께 사용한다. 어떤 클라이언트가 영향을 받는지 모른 채 전체 강제하면 인증이 끊기는 서비스가 생길 수 있다.

## 원본 하드닝 표를 연결 방식별로 나눈다

| 연결·인증 | 보호 판단 | 시험할 것 |
|---|---|---|
| TLS 없는 Simple Bind | 자격 증명 전송 보호가 없음 | 전환 대상 식별, 운영 자격 증명으로 재현하지 않음 |
| TLS 없는 SASL | 협상된 서명·sealing에 따라 달라짐 | 실제 서명/암호화 협상과 서버 요구 정책 |
| TLS 위 Simple Bind | TLS의 신뢰·이름 검증이 중요 | 인증서 검증을 켠 실제 클라이언트로 Bind·Search |
| TLS 위 SASL | TLS와 인증, 필요 시 CBT를 함께 검사 | 클라이언트 CBT 지원·서버 요구·중간 TLS 종료 |

LDAP 서명은 TLS 없는 SASL 메시지의 무결성과 연결되고, TLS 연결에는 TLS 무결성이 적용된다. CBT는 TLS 세션과 SASL 인증을 묶는다. 모든 Simple Bind에 같은 방식으로 CBT가 적용된다고 설명하지 않는다. [LDAP 보호 계층](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing), [CBT 적용 연결](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-channel-binding)

원본 LDS 문서의 LDAPS 활성화 절차와 AD DS 정책은 같은 기능이라고 합치지 않는다. 서비스별 인증서 선택·개인키 접근·신뢰·포트와 인증 방식·정책을 각각 기록한다. 로드밸런서·프록시가 TLS를 종료하면 클라이언트와 DC가 보는 채널이 달라질 수 있어 CBT 호환성도 확인한다.

## 감사에서 강제까지의 판정표

| 단계 | 확인 | 중단·원복 조건 |
|---|---|---|
| 현황 | 클라이언트·라이브러리·OS·인증·TLS·정책·담당자 | 사용처·책임자가 불명확하면 강제 확대 보류 |
| 감사 | 대상 DC의 지원 감사 이벤트와 앱 실패·접속 시각 | 수집 누락을 비호환 클라이언트 없음으로 판정하지 않음 |
| 파일럿 | 정상·비정상 인증서, 지원·미지원 인증 방식 | 승인 앱 기능 실패 또는 관리 접근 손실 |
| 단계 강제 | DC별 유효 정책과 허용/거부 결과 | 예상 밖 인증 실패 시 이번 정책 변경 범위 복원 |
| 안정화 | 예외 종료·클라이언트 개선·재검사 | 보안 완화 예외를 기한 없이 유지하지 않음 |

LDAP 서명 관련 2886–2889와 CBT 관련 3039–3041 등은 OS·업데이트·진단 설정과 이벤트 의미를 확인해 사용한다. 요약 이벤트와 개별 클라이언트 이벤트는 다르며 모든 이벤트가 모든 버전에 같은 조건으로 기록되는 것은 아니다. [서명 모니터링](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing), [CBT 모니터링·시행](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-channel-binding)

인증서 검증을 끄거나 모든 LDAP 보호를 완화하는 것을 표준 원복으로 제시하지 않는다. 기록한 유효 정책 중 변경분과 필요한 파일럿 범위만 조정하고 원인을 조사한다. TLS 인증서 변경과 서명·CBT 정책을 한 번에 바꾸면 실패 원인과 원복 범위를 구분하기 어려워진다.

## 신규 설치와 업그레이드를 구분한다

LDAP 보안 기본값은 Windows Server 버전과 신규 배포·업그레이드 여부에 따라 다를 수 있다. 현재 공식 문서는 Windows Server 2025 신규 AD 배포의 서명 기본 강화를 설명한다. OS 이름만 보고 기존 정책까지 동일하다고 가정하지 않는다. [Microsoft 버전별 LDAP 기본 동작](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing)

감사·호환성 확인, 파일럿, 단계 적용, 인증 실패 모니터링 순으로 진행한다. 원복하더라도 예외 대상과 재적용 조건을 남겨야 보안 완화를 영구 설정으로 굳히지 않을 수 있다.

---

적용 기준: Windows Server 2019·2022·2025의 AD DS. 신규 배포·업그레이드 차이는 본문 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.

[연재 목차]({% post_url 2026-08-13-ad-series-guide %})
