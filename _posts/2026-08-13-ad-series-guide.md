---
layout: single
title: "Active Directory 심화·보안 연재: 설계·검증·운영의 판단 기준"
categories: [Active Directory]
tags: [AD, AD LDS, Windows Server, Security, Portfolio]
toc: true
toc_sticky: true
published: false
---

AD 구축·DNS·계정과 권한·보안·운영·AD LDS 업무를 설계 이유와 검증 기준으로 연결한 연재다. 원본 구축·DNS·보안·LDS 문서의 표와 화면을 선별해, 구성 요소의 관계·선택의 비용·확인 방법·장애 분기·변경 및 원복 판단을 보강했다.

읽는 순서는 기초→DNS→인증·권한·정책→인프라→보안·운영→LDS다. 특정 장애를 조사한다면 관련 글의 관찰·판정표부터 읽어도 된다. 각 글은 실행 위치·권한과 결과의 한계를 설명하며 실제 환경의 승인된 Runbook을 대신하지 않는다.

## 기초와 DNS

| 편 | 글 | 중심 질문 |
|---|---|---|
| 01 | [AD DS 운영 구조]({% post_url 2026-08-17-ad-overview %}) | 인증·권한·정책과 복구 책임은 어디에 있는가? |
| 02 | [Forest·Domain·OU 설계]({% post_url 2026-08-20-ad-forest-domain-ou %}) | 위임과 보안 경계를 어떻게 선택하는가? |
| 03 | [객체 식별과 스키마]({% post_url 2026-08-24-ad-objects-schema-dn %}) | OU 이동·객체 재생성이 연동에 미치는 영향은? |
| 04 | [DC Locator 진단]({% post_url 2026-08-27-ad-dns-dc-discovery %}) | 레코드·후보 응답·Site·실제 통신을 어떻게 나누는가? |
| 05 | [AD 통합 DNS 설계]({% post_url 2026-08-31-ad-dns-integrated-zones %}) | 저장·복제 범위·갱신 권한을 어떻게 검증하는가? |
| 06 | [DNS 전달 설계]({% post_url 2026-09-04-ad-dns-forwarders %}) | 질의 경로·실패 응답·대체 경로를 어떻게 확인하는가? |

## 인증·권한·정책

| 편 | 글 | 중심 질문 |
|---|---|---|
| 07 | [Kerberos 심화]({% post_url 2026-09-07-ad-kerberos-flow %}) | 티켓·SPN·시간·감사 로그에서 실패 단계를 찾는가? |
| 08 | [LDAP 연동 진단]({% post_url 2026-09-11-ad-ldap-and-kerberos %}) | 연결·TLS·Bind·Search·ACL은 각각 성공했는가? |
| 09 | [도메인 가입 심화]({% post_url 2026-09-14-ad-domain-join %}) | 객체 재사용과 보안 채널·업무 전환을 검증했는가? |
| 10 | [OU·위임 설계]({% post_url 2026-09-18-ad-ou-delegation %}) | 관리 그룹·상속·로그온 경로를 제한했는가? |
| 11 | [AGDLP 운영]({% post_url 2026-09-22-ad-groups-agdlp %}) | 직무 이동 후 이전 접근이 실제로 회수됐는가? |
| 12 | [GPO 설계·배포]({% post_url 2026-09-25-ad-gpo-processing %}) | 우선순위·필터·예외와 원복을 함께 설계했는가? |
| 13 | [GPO 진단]({% post_url 2026-09-29-ad-gpo-troubleshooting %}) | 대상·AD 정보·SYSVOL·실제 설정을 어떻게 추적하는가? |

## 인프라·보안·운영

| 편 | 글 | 중심 질문 |
|---|---|---|
| 14 | [복제 장애 진단]({% post_url 2026-10-02-ad-replication %}) | 어느 파티션·파트너가 언제부터 실패했는가? |
| 15 | [Site 물리 설계]({% post_url 2026-10-06-ad-sites-subnets %}) | Subnet·WAN·장애 도메인·통신 구간이 맞는가? |
| 16 | [GC·FSMO·기능 수준]({% post_url 2026-10-10-ad-gc-fsmo %}) | 역할 배치와 이전·강제 점유를 어떻게 판단하는가? |
| 17 | [관리자 보안과 탐지]({% post_url 2026-10-13-ad-privileged-access %}) | 관리 경로와 관제 수집 공백을 통제했는가? |
| 18 | [Windows LAPS 설계]({% post_url 2026-10-17-ad-windows-laps %}) | 회전·저장·조회·복호화가 각각 검증됐는가? |
| 19 | [gMSA 전환]({% post_url 2026-10-20-ad-service-accounts-gmsa %}) | KDS·호스트·서비스·업무 권한을 나눠 확인했는가? |
| 20 | [LDAP 하드닝]({% post_url 2026-10-24-ad-ldap-security %}) | 바인드 방식별 보호와 호환성을 조사했는가? |
| 21 | [운영 점검 체계]({% post_url 2026-10-28-ad-health-troubleshooting %}) | 주기·관찰·경보·조치와 담당자가 연결되는가? |
| 22 | [백업·복구 설계]({% post_url 2026-10-31-ad-backup-recovery %}) | 장애 범위에 맞는 복구 지점과 업무 검증이 있는가? |

## AD LDS 후속 3편

| 편 | 글 | 중심 질문 |
|---|---|---|
| LDS 01 | [도입·구조 설계]({% post_url 2026-11-04-ad-lds-architecture %}) | 인스턴스·파티션·포트·운영 책임은 어떻게 나누는가? |
| LDS 02 | [구성·보안·복제]({% post_url 2026-11-07-ad-lds-deployment-security %}) | 설치에서 TLS·Bind·권한·복구까지 확인했는가? |
| LDS 03 | [ADAMSync·userProxy]({% post_url 2026-11-11-ad-lds-sync-auth %}) | 객체 동기화·암호·인증 연동·SSO를 구분했는가? |

## 예제·원본 화면·실행 결과를 구분한다

`ad.example.com`, `partner.example.net`, `DC01`·`DC02`·`LDS01`, `user01`, `192.0.2.53` 등은 설명용 가상 값이다. 본문의 명령에 실제 출력처럼 꾸민 값을 붙이지 않았다. 원본 화면은 앞뒤 문맥과 대조했고 회사·도메인·서버·계정·주소·경로 등 식별정보가 있는 영역을 가렸다. 설명용 도식은 실습 성공 증적으로 사용하지 않는다.

실행 결과를 보여주는 원본 화면은 **그 화면에서 확인되는 항목**만 설명한다. AD 통합 영역 출력은 전체 복제 정상, LDIF 가져오기는 앱 호환성, ADAMSync 설정 설치는 객체 동기화, 포트 연결은 TLS·Bind 성공을 각각 대신하지 않는다.

각 글의 근거는 Microsoft 공식 문서의 본문·조건과 연결했다. 이전 버전 문서는 현재 OS의 설치·지원 조건과 구분한다. 이번 작성에서는 실제 AD/DNS/LDS 명령 실행, 장애 재현, 보안 정책 적용, 클라우드 운영, 복구 훈련을 새로 수행하지 않았고 그 성과를 만들어 넣지 않았다.

## 별도 주제로 남긴 범위

이 연재의 25편 외에 전체 관제 구현 코드, DNSSEC 상세 운영, JML 자동화, Forest Trust 실습, AD CS, 대규모 이관, Microsoft Entra 하이브리드 연동은 후속 주제로 남겼다. 각 글에서는 관련 설계·판단 기준을 설명하고 실제 수행 이력과 구분했다.

---

원본·공식 근거 검토 기준일: 2026-11-10. 버전별 적용 조건과 검증의 한계는 각 글 참조.
