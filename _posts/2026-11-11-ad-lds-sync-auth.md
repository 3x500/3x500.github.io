---
layout: single
title: "AD LDS 03. ADAMSync·userProxy: 객체 동기화와 인증·SSO 구분하기"
categories: [Active Directory]
tags: [AD LDS, ADAMSync, userProxy, LDAP, Authentication]
toc: true
toc_sticky: true
published: false
---

AD DS 사용자를 LDS에 복사했다고 도메인 암호와 자동 로그인까지 복사된 것은 아니다. 원본 LDS 문서의 동기화·앱 연동·고유 계정 절을 **데이터 전달, Bind 인증, 앱의 로그인 경험, 권한 부여**로 나눠 해석한다.

## 서로 다른 세 기능

| 구분 | 전달·처리하는 것 | 별도로 검증할 것 |
|---|---|---|
| ADAMSync | 지정 원본의 객체·속성을 대상 LDS에 동기화 | 필터·속성·변경·삭제·범위·대상 스키마 |
| LDS 고유 사용자 | LDS 계정의 자격 증명으로 인증 | 계정 상태·암호 정책·TLS·ACL |
| userProxy / Bind Proxy | Windows 보안 주체를 참조하는 객체를 통한 인증 | SID 매핑·해당 Windows 신원·지원 Bind·LDS 권한 |

Bind Proxy는 그 자체가 Windows 보안 주체가 아니며 암호를 저장하지 않는다. `objectSid`가 참조하는 Windows 사용자와 신뢰 범위 조건이 중요하다. “AD 암호가 LDS에 복제돼 저장된다”는 설명으로 옮기지 않는다. [Bind Proxies의 공식 동작](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/2ed97617-00a9-4f8c-8005-23a5f926111f)

## 원본 ADAMSync 절차를 설정·실행·결과로 나눈다

```text
AD DS의 승인 원본 범위·읽기 신원
    → ADAMSync의 범위·속성 설정과 실행 상태
    → LDS 앱 파티션·스키마·쓰기 권한
    → 앱의 Search·신원 매핑·권한 검사
```

원본은 스키마 비교·준비 후 `MS-AdamSyncConf.xml`을 바탕으로 별도 XML을 만들고, 설정 설치와 동기화를 나눠 수행한다. 출발점·대상 Naming Context·기존 데이터 소유권을 먼저 정한다. LDS에서 따로 수정한 속성을 다음 실행이 덮어쓰는지도 확인한다.

| XML·운영 항목 | 확인할 질문 |
|---|---|
| `source-ad-name` | 실제 읽을 AD DS와 도달·인증 경로는 무엇인가? |
| `source-ad-partition` | 원본 Naming Context와 일치하는가? |
| `source-ad-account`, `account-domain` | 실제 실행 신원·권한은 무엇이며 비밀값을 어떻게 보호하는가? |
| `target-dn` | 대상 앱 파티션의 Head인가? |
| `base-dn`, `object-filter` | 의도한 객체만 포함하고 필요한 상위 구조가 준비됐는가? |
| `include`·`exclude` 속성 | 대상 스키마·앱 요구·민감 속성과 일치하는가? |
| 실행 주기·상태·로그 | 증분·삭제·중단·재시도 결과를 어떻게 추적하는가? |

실제 XML은 설치된 버전의 템플릿을 기준으로 검토한다. 문법이 맞는 XML과 제품이 허용하는 구조·검색 범위는 다른 검사다. 최초 동기화에서 모든 속성을 무조건 가져온 뒤 줄이는 방법을 표준으로 쓰지 않는다. 필요한 객체·속성의 파일럿 범위를 먼저 정한다. [ADAMSync 준비·설정·실행 구분](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2008-r2-and-2008/cc770408(v=ws.10))

![원본 ADAMSync 설정 설치 로그. 서버·파일 경로·대상 DN을 가림]({{ '/images/ad-deep-dive/d-042.png' | relative_url }})

원본 화면의 Saved configuration file·Done은 **설정 설치 로그**다. 동기화 실행·객체 수렴·삭제 반영·인증 성공을 증명하지 않는다. 명령을 입력한 화면도 실행 성공 증적으로 대체하지 않는다.

## 동기화 검증은 생성 외의 변경도 포함한다

| 테스트 | 기대 관찰 | 실패 시 다음 확인 |
|---|---|---|
| 범위 안 테스트 객체 생성 | 대상에 기대 클래스·속성으로 생성 | Base DN·필터·대상 스키마·쓰기 권한 |
| 속성 수정 | 허용 속성만 갱신 | include/exclude·값 형식·실행 상태 |
| 범위 밖·제외 속성 | 승인 범위 밖 데이터는 전달되지 않음 | 필터 논리·상위 구조·속성 목록 |
| 객체 이동·이름 변경·삭제 | 승인된 처리 규칙과 일치 | 원본 범위 이탈·대상 소유권·삭제 설정·앱 참조 |
| 중단·재실행 | 누락·중복·상태 오류를 추적 | 실행 로그·체크포인트·동시 실행 |

동기화가 끝나지 않는다고 단순히 기다리거나 계속 재실행하지 않는다. 공식 문서에는 손자 OU를 Base DN으로 지정한 ADAMSync의 알려진 문제가 있다. 해당 버전·구조·기존 대상 트리 조건과 로그를 대조한다. 우회하려고 검색 범위를 확대하면 데이터 노출·삭제 영향이 달라질 수 있으므로 다시 승인·검증한다. [Base DN 관련 알려진 문제](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/adamsync-cant-sync-basedn-grandchild-ou)

## userProxy 인증과 SSO를 구분한다

```text
앱 → TLS로 보호된 LDAP Simple Bind → LDS의 Bind Proxy 객체
                                    → objectSid가 참조하는 Windows 신원 검증
성공한 LDAP 신원 → LDS ACL 및 앱 권한 검사
```

userProxy는 Bind Proxy 구현을 위한 클래스다. Bind Proxy의 지원 인증 방식은 Simple Bind이며, Windows 계정의 SASL 인증이나 LDS 고유 사용자 인증과 구분한다. TLS 보호와 서버 인증서 검증, 계정 형식, 참조 SID·신뢰·도달 조건을 함께 시험한다. [userProxy 클래스](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adls/bce5204d-3caf-499e-88c5-b209b825c7cf), [LDS 계정 유형별 Bind 지원](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/89ea294a-5b65-47bc-8ed5-51cc925858a7)

암호를 앱이 다시 입력받아 Simple Bind를 수행하는 흐름은 사용자에게 자격 증명 재입력이 없는 SSO와 같은 기능이 아니다. 원본의 “userProxy SSO” 표현은 인증 연동으로 범위를 수정했다. 실제 무입력 SSO가 목표라면 앱이 지원하는 통합 인증·세션·위임 등 별도 흐름과 보안 조건을 검증한다.

AD DS 계정을 삭제 후 같은 이름으로 재생성하면 SID가 달라질 수 있다. 기존 proxy가 새 계정을 이름만으로 자동 참조한다고 가정하지 않는다. AD 계정 비활성화·암호 변경과 앱의 기존 세션·캐시 종료도 같은 시점에 일어난다고 보지 않는다.

## LDS 고유 계정과 전환·원복

원본 부록의 고유 사용자 암호 설정·활성화·Simple Bind는 별도 계정 경로다. TLS 보호가 확인되지 않은 연결·Bind 결과를 보안 인증 완료로 해석하지 않는다. 암호 설정 성공, 계정 활성, TLS·Bind, 객체 읽기, 앱 권한을 각각 확인한다.

변경 전 XML·템플릿 버전·원본/대상 범위·스키마·기존 객체·실행 신원·백업을 기록한다. 과다 생성·수정·삭제 또는 인증 실패가 있으면 스케줄을 중단하고 이번 실행의 로그와 변경 범위를 보존한다. 설정 파일을 이전 버전으로 되돌리는 것만으로 이미 삭제·수정된 데이터가 복구되지는 않는다. 대상 복원·재동기화 기준을 정하고 앱의 검색·SID 매핑·허용/거부·기존 세션을 재검증한다.

---

원본 D 동기화·인증 연동과 공식 프로토콜·진단 본문 대조: 2026-11-10. 원본 결과와 기대 검증을 구분했다. 이번 작성에서 ADAMSync·Bind·계정 활성화·TLS 검증을 실행하지 않았다.

[연재 목차]({% post_url 2026-08-13-ad-series-guide %})
