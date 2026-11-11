---
layout: single
title: "AD LDS 02. 구성·보안·복제: 설치 완료에서 앱 검증까지"
categories: [Active Directory]
tags: [AD LDS, LDAPS, Schema, Replication, Backup]
toc: true
toc_sticky: true
published: false
---

AD LDS 역할 설치와 앱이 사용할 디렉터리 준비는 다른 단계다. 원본 LDS 문서의 설치→인스턴스→스키마→연결→보안→복제·복구 순서를 따라 각 단계에서 무엇을 확인해야 다음으로 넘어갈 수 있는지 정리한다.

설명용 값은 인스턴스 `AppDirectory`, 서버 `LDS01.ad.example.com`, LDAP 50389·TLS 50636, 앱 파티션 `DC=app,DC=example,DC=com`이다. 원본 화면의 식별정보는 가렸으며 실제 원본 환경과 같은 이름·포트라고 가정하지 않는다.

## 설치·인스턴스·서비스를 나눠 확인한다

| 단계 | 원본 절차를 반영한 작업 | 성공 기준 |
|---|---|---|
| 역할 | Server Manager에서 AD LDS와 관리 도구 준비 | 역할 설치 상태. 아직 앱 디렉터리 정상 판정은 아님 |
| 인스턴스 | 새 구성 집합 또는 기존 복제본 선택 | 선택한 집합·서비스·포트·경로·계정이 설계와 일치 |
| 파티션 | 앱 Naming Context 생성·선택 | RootDSE·트리에서 필요한 파티션 확인 |
| 스키마 | 앱에 필요한 LDIF만 검토·적용 | 대상 집합의 클래스·속성과 로그 확인 |
| 접속 | 앱과 같은 Endpoint·보호·신원으로 시험 | TLS→Bind→Search→권한을 각각 검사 |

역할 조회는 해당 Windows Server의 관리자 PowerShell에서 ServerManager 모듈로 수행할 예제다. 실제 설치·실행 결과는 만들지 않았다.

```powershell
Get-WindowsFeature -Name ADLDS
Get-Service -Name 'ADAM_AppDirectory'
```

서비스 이름은 설치 결과를 확인해 바꾼다. `Installed`·`Running`과 포트 리슨은 준비 상태이며 TLS·객체·권한·앱 업무 성공을 증명하지 않는다. Microsoft의 현재 AD LDS 테스트 준비 안내도 역할과 인스턴스 생성을 구분한다. [역할·인스턴스·TLS 테스트 준비](https://learn.microsoft.com/en-us/entra/identity/app-provisioning/on-premises-ldap-connector-prepare-directory)

## LDIF 가져오기는 대상과 변경 범위를 검토한다

원본은 ADSchemaAnalyzer로 원본·대상 스키마를 비교하고 생성한 LDIF의 Naming Context를 대상 LDS에 맞춘 뒤 가져온다. 클래스·속성·OID·구문·필수 속성·기존 정의·치환 대상을 검토한다. AD DS의 모든 스키마를 의미 확인 없이 복사하는 작업으로 일반화하지 않는다.

![원본 LDS 스키마 LDIF 가져오기 결과. 서버·파일 경로와 명령 영역은 가림]({{ '/images/ad-deep-dive/d-036.png' | relative_url }})

원본 화면에는 335 entries modified successfully와 명령 완료가 표시된다. 이는 해당 원본의 **가져오기 실행 결과**이며 현재 테스트나 앱 호환성·TLS·복제 성공 증적이 아니다. 명령의 `-k`처럼 일부 오류를 무시할 수 있는 옵션이 쓰였다면 완료 문장 외에 상세 로그와 기대 정의도 대조한다.

스키마 추가를 일반 객체 삭제처럼 원복할 수 있다고 가정하지 않는다. 테스트→검증→운영의 단계, 변경 LDIF 버전, 백업·복원 조건과 앱 영향을 먼저 정한다. 기본 사용자 클래스 LDIF도 앱이 요구하는 유형만 선택한다. [AD LDS 사용자 클래스 가져오기](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/adam/adding-user-classes)

## TLS와 Bind·권한의 완료 조건

| 항목 | 확인 | 실패 시 분기 |
|---|---|---|
| 인증서 | 앱이 쓰는 FQDN·서버 인증 용도·유효기간·체인·개인키 | 이름·신뢰·선택 저장소·키 누락 |
| 서비스 계정 | 선택 인증서의 필요한 키 읽기 | 실제 서비스 신원·개인키 ACL |
| TLS 연결 | 앱 클라이언트에서 지정 TLS 포트·인증서 검증 | 포트·리스너·암호군·라이브러리·TLS 종료 |
| Bind | 지원 인증 방식·계정 유형·상태 | TLS 성공과 자격 증명 검증 분리 |
| Search·ACL | Base DN·범위·필터·속성·허용 작업 | 객체 위치·스키마·읽기/쓰기 권한 |

인증서를 설치했다는 사실만으로 LDAPS가 검증되지는 않는다. 서비스 저장소·컴퓨터 저장소, 실제 선택 인증서와 서비스 계정의 개인키 접근을 확인하고 LDP의 FQDN·TLS 포트·SSL 옵션 또는 실제 앱으로 검사한다. 이전 버전 안내의 UI·키 경로는 최신 암호화 공급자·OS 조건에 맞춰 확인한다. [AD LDS 인증서·계정 권한·연결 요구](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2008-r2-and-2008/cc725767(v=ws.10))

앱 조회 계정에 인스턴스 관리자 역할을 주는 것은 최소 권한 검증이 아니다. 필요한 파티션·속성 읽기와 비승인 쓰기 거부를 별도로 확인한다. 원본의 “AD LDS에 FGPP 적용” 설명은 AD DS 도메인 FGPP와 같은 기능으로 옮기지 않았다. AD LDS 계정의 암호·잠금 정책은 계정 유형·호스트 정책·LDS 설정의 실제 지원 조건으로 별도 검토한다. [AD LDS 암호 제약](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/7dfeb38c-3cb9-4215-ae1c-ef209fd251ae), [FGPP 명령의 AD LDS 미지원](https://learn.microsoft.com/en-us/powershell/module/activedirectory/set-adfinegrainedpasswordpolicy)

## 복제본을 추가한 뒤 같은 객체를 비교한다

원본의 복제본 설치 절차에서는 기존 구성 집합·원본 서버와 앱 파티션을 선택한다. 구성·스키마와 앱 파티션 참여 범위를 확인하고 각 서버의 서비스 계정·인증·연결 포트를 기록한다. 같은 서버의 다중 인스턴스를 서버 이중화로 세지 않는다.

관리 도구와 조회 권한이 있는 호스트에서 LDS 인스턴스의 실제 LDAP 포트를 지정해 상태를 읽는다.

```text
repadmin /showrepl LDS01.ad.example.com:50389
```

파트너·Naming Context·마지막 성공·오류와 동일 테스트 객체의 값을 양쪽에서 비교한다. 포트를 생략해 다른 디렉터리를 조회하지 않도록 주의한다. 강제 복제는 이 글의 조회 예제에 넣지 않았다. 인증 수준을 낮추기 전에 서비스 신원·SPN·DNS·시간·네트워크와 현재 복제 인증 요구를 확인한다. [AD LDS 복제 인증 수준](https://learn.microsoft.com/en-us/dotnet/api/system.directoryservices.activedirectory.replicationsecuritylevel)

## 복제와 백업·복구를 분리한다

| 준비 | 기록할 것 | 복구 검증 |
|---|---|---|
| 앱 데이터·구성 | 인스턴스·파티션·DB/로그·버전·스키마 | 객체·속성·ACL·앱 참조 |
| TLS·서비스 신원 | 인증서·개인키 보관·권한·서비스 설정 | 이름·신뢰·키 접근·실제 Bind |
| 독립 백업 | 도구·지원 OS·일관성·보관·복구 지점 | 격리된 복원과 사용 가능한 백업 |
| 복제 집합 복원 | 정상 기준과 권위/비권위 판단 | 복원한 변경이 재삭제·재오염되지 않는지 |

원본의 dsdbutil·IFM 언급을 전체 복원용 백업과 동일하게 취급하지 않는다. 과거 공식 문서의 NTBackup·테이프 UI를 Windows Server 2019 이상의 검증된 Runbook으로 복사하지 않았다. 백업 도구 지원과 인스턴스 일관성을 확인한 복원 훈련이 필요하다. [인스턴스 복원과 권위 복원 개념](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/adam/restoring-an-instance)

변경 후 앱 기능·권한·복제·재시작을 확인하고 실패하면 확대를 중단한다. 스키마·DB 복원은 설정 몇 개를 역순으로 바꾸는 원복과 구분한다. 복구 성공은 서비스 시작과 같은 객체·권한·업무 결과를 모두 확인한 시점으로 정한다.

---

원본 D 설치·스키마·보안·복제·복구 장과 공식 본문 대조: 2026-11-10. 이전 버전 근거는 개념과 조건을 구분해 사용했다. 이번 작성에서 설치·LDIF·TLS·복제·복원을 실제 실행하지 않았다.

[연재 목차]({% post_url 2026-08-13-ad-series-guide %})
