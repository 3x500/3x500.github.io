---
layout: single
title: "AD 20. LDAP 서명·TLS·채널 바인딩의 차이"
categories: [Active Directory]
tags: [AD, LDAP Signing, TLS, Channel Binding]
toc: true
published: false
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

## 신규 설치와 업그레이드를 구분한다

LDAP 보안 기본값은 Windows Server 버전과 신규 배포·업그레이드 여부에 따라 다를 수 있다. 현재 공식 문서는 Windows Server 2025 신규 AD 배포의 서명 기본 강화를 설명한다. OS 이름만 보고 기존 정책까지 동일하다고 가정하지 않는다. [Microsoft 버전별 LDAP 기본 동작](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing)

감사·호환성 확인, 파일럿, 단계 적용, 인증 실패 모니터링 순으로 진행한다. 원복하더라도 예외 대상과 재적용 조건을 남겨야 보안 완화를 영구 설정으로 굳히지 않을 수 있다.

---

적용 기준: Windows Server 2019·2022·2025의 AD DS. 신규 배포·업그레이드 차이는 본문 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.
