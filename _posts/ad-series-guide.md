---
layout: single
title: "Active Directory 운영 개념 정리: 연재 안내"
categories: [Active Directory]
tags: [AD, Windows Server, Portfolio]
toc: true
published: false
---

AD 구축·DNS·계정과 권한·보안·운영 점검 업무를 설명하기 위해 정리한 연재다. 설정 순서뿐 아니라 구성 이유, 정상 여부를 확인하는 방법, 장애 범위를 좁히는 기준을 다룬다.

기초부터 읽는다면 아래 순서를 권장한다. 운영 중 궁금한 주제가 있다면 해당 글부터 읽어도 된다.

## 기초와 DNS

| 번호 | 글 | 다루는 질문 |
|---|---|---|
| 01 | [Active Directory는 무엇을 관리하는가](ad-overview.md) | 중앙 계정 관리가 필요한 이유는 무엇인가? |
| 02 | [Forest·Domain·OU는 어떻게 다른가](ad-forest-domain-ou.md) | 구조마다 나누는 범위와 목적은 무엇인가? |
| 03 | [객체·속성·스키마·DN 이해하기](ad-objects-schema-dn.md) | 디렉터리의 정보를 어떻게 구분하는가? |
| 04 | [AD는 DNS로 DC를 어떻게 찾는가](ad-dns-dc-discovery.md) | SRV와 DC 검색은 어떻게 연결되는가? |
| 05 | [AD 통합 DNS 영역과 동적 업데이트](ad-dns-integrated-zones.md) | 레코드를 어디에 저장하고 누가 갱신하는가? |
| 06 | [Forwarder와 Conditional Forwarder](ad-dns-forwarders.md) | 어떤 질의를 어느 DNS로 전달하는가? |

## 인증과 관리

| 번호 | 글 | 다루는 질문 |
|---|---|---|
| 07 | [Kerberos 로그인과 티켓의 흐름](ad-kerberos-flow.md) | TGT와 서비스 티켓은 어떻게 쓰이는가? |
| 08 | [LDAP와 Kerberos의 역할 차이](ad-ldap-and-kerberos.md) | 조회와 인증은 어떻게 연결되는가? |
| 09 | [도메인 가입과 컴퓨터 계정](ad-domain-join.md) | PC의 도메인 참여는 무엇을 의미하는가? |
| 10 | [OU 설계와 관리 권한 위임](ad-ou-delegation.md) | 관리 책임을 어떻게 나누는가? |
| 11 | [보안 그룹과 AGDLP](ad-groups-agdlp.md) | 역할과 리소스 권한을 어떻게 연결하는가? |
| 12 | [GPO 적용 순서와 적용 범위](ad-gpo-processing.md) | 어느 정책이 어떤 대상에 적용되는가? |
| 13 | [GPO가 적용되지 않을 때 확인할 것](ad-gpo-troubleshooting.md) | 대상 문제와 처리 실패를 어떻게 구분하는가? |

## 인프라·보안·운영

| 번호 | 글 | 다루는 질문 |
|---|---|---|
| 14 | [DC 복제와 데이터 일관성](ad-replication.md) | DC마다 결과가 다르면 무엇을 확인하는가? |
| 15 | [Site·Subnet과 DC 선택](ad-sites-subnets.md) | 지점 PC의 위치와 DC 검색은 어떤 관계인가? |
| 16 | [GC와 FSMO 역할 이해하기](ad-gc-fsmo.md) | 검색 기능과 특정 작업 책임은 어떻게 다른가? |
| 17 | [관리자 계정 분리와 최소 권한](ad-privileged-access.md) | 권한과 관리 경로를 어떻게 제한하는가? |
| 18 | [Windows LAPS의 목적과 운영](ad-windows-laps.md) | 장비별 로컬 관리자·DSRM 암호를 어떻게 관리하는가? |
| 19 | [서비스 계정과 gMSA](ad-service-accounts-gmsa.md) | 서비스 신원과 암호를 어떻게 관리하는가? |
| 20 | [LDAP 서명·TLS·채널 바인딩](ad-ldap-security.md) | 보호 범위와 호환성 조건은 무엇인가? |
| 21 | [AD 일상 점검과 장애 진단 순서](ad-health-troubleshooting.md) | 어떤 근거로 원인 범위를 좁히는가? |
| 22 | [DC 이중화·백업·복구의 차이](ad-backup-recovery.md) | 서비스 지속과 과거 상태 복구는 어떻게 다른가? |

## 예제와 근거

도메인 `ad.example.com`, 별도 이름 공간 `partner.example.net`, 서버 `DC01`·`DC02`, 계정 `user01` 등은 설명용 가상 값이다. 실제 회사명·담당자·운영 주소·암호·내부 화면은 포함하지 않았다.

AD DS 설명은 Windows Server 2019 이상 환경의 개념·운영을 중심으로 한다. 특정 버전·기능 수준·업데이트가 필요한 기능은 각 글에서 구분한다. PowerShell 예제는 필요한 관리 모듈과 권한을 확인하고 사용해야 한다.

각 글의 기술적 근거는 Microsoft 공식 문서에 연결했다. 명령어는 조회·진단 방법을 설명하는 예제이며, 이번 글 작성 과정에서 실제 AD 환경의 실행 결과를 수집하지 않았다. 실제 장애 해결 이력이나 성과 수치를 새로 만들어 넣지 않았다.

AD LDS, Forest Trust, Microsoft Entra ID, AD CS는 핵심 연재 이후 별도 주제로 확장한다.

---

공식 근거 검토일: 2026-11-10.

<!-- 작성자용: 현재 목차 링크는 로컬 Markdown 검토용이다. 게시 승인 후 날짜가 확정된 _posts 파일로 이동하고 실제 게시 URL 또는 Jekyll post_url로 변경한다. -->
