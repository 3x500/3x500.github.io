---
layout: single
title: "AD 13. GPO 진단: 대상 선정·GPC/GPT·복제·실제 설정 추적"
categories: [Active Directory]
tags: [AD, GPO, gpresult, Troubleshooting]
toc: false
toc_sticky: true
published: true
---

정책이 적용되지 않으면 `gpupdate /force`부터 반복하기 쉽다. 하지만 새로 고침만으로 잘못된 링크나 필터 조건을 해결할 수는 없다. 먼저 대상 선정 문제와 정책 처리 실패를 구분한다.

택배에 비유하면 주소가 다른 것과 배송 중 문제가 생긴 것은 다르다. 대상에 포함되지 않은 GPO와, 대상이지만 처리에 실패한 GPO를 나누어 봐야 한다. [Microsoft GPO 문제 해결](https://learn.microsoft.com/en-us/troubleshoot/windows-server/group-policy/applying-group-policy-troubleshooting-guidance)

## 확인할 순서

1. 사용자·컴퓨터 객체가 어느 OU에 있는지 확인한다.
2. 해당 경로의 링크 활성 상태와 사용자·컴퓨터 설정 상태를 확인한다.
3. 보안·WMI 필터와 상속·강제 적용 조건을 확인한다.
4. 클라이언트가 사용한 DC와 정책 결과를 확인한다.
5. 처리 실패라면 GroupPolicy 로그와 DNS·SYSVOL·복제를 확인한다.

## 결과 보고서 만들기

영향받는 PC에서 확인한다. 컴퓨터 범위 보고서는 관리자 권한으로 실행하고, 사용자 범위는 문제를 겪는 사용자 세션을 기준으로 확인한다.

```text
gpresult /scope computer /r
gpresult /scope user /r
gpresult /h "%TEMP%\ad-gpresult.html"
```

마지막 명령은 보고서 파일을 생성한다. 이 글은 명령 프롬프트 구문이며 실제 보고서를 첨부하지 않았다. 관리자 계정을 다른 사용자로 실행하면 대상 사용자와 보고서의 사용자 범위가 달라질 수 있으므로 계정을 함께 확인한다. [Microsoft gpresult](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/gpresult)

## 보고서를 읽을 때 보는 것

| 관찰 | 다음 확인 |
|---|---|
| 원하는 GPO가 적용 목록에 없음 | OU·링크·필터·권한 확인 |
| GPO는 적용됐지만 설정이 다름 | 항목별 최종 적용 GPO와 우선순위 확인 |
| 특정 PC·지점에서만 실패 | DC·DNS·SYSVOL 접근·복제 상태 비교 |
| 오류 이벤트가 있음 | 같은 시각·처리 작업의 오류 내용 확인 |

GPO가 목록에 있다는 사실만으로 모든 항목이 기대대로 처리됐다고 보지 않는다. 설정에 따라 재시작·로그오프가 필요할 수 있고, 클라이언트 확장 처리 결과도 확인해야 한다. [Microsoft GPO 처리 방식](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-policy/group-policy-processing)

## GPC와 GPT를 나눠 원인을 좁힌다

GPO에는 AD의 Group Policy Container(GPC) 정보와 SYSVOL의 Group Policy Template(GPT) 파일이 관련된다. GPC의 AD 복제와 SYSVOL의 파일 복제 상태는 별도다. AD 복제 요약이 정상이어도 정책 파일 접근이나 파일 복제 문제가 남을 수 있다. [GPO 처리 실패와 의존 구성요소 진단](https://learn.microsoft.com/en-us/troubleshoot/windows-server/group-policy/applying-group-policy-troubleshooting-guidance)

| 단계 | 관찰 자료 | 판정·다음 확인 |
|---|---|---|
| 대상 선정 | 객체 DN, 링크·필터·권한, 보고서 제외 이유 | 대상 밖인지 처리 실패인지 분리 |
| DC 선택 | 문제 단말이 사용한 DC·DNS·시각 | 특정 DC·Site만 다른지 비교 |
| AD 정보 | GPO GUID·설정·버전·링크 | 다른 DC의 정보와 해당 파티션 복제 |
| SYSVOL | 해당 GUID의 정책 파일 경로·읽기·버전 | SMB·권한·DFSR·파일 누락 |
| 클라이언트 확장 | 동일 처리의 Activity ID와 시작·종료·오류 | 설정 확장·동기/비동기·재시작 요구 |
| 실제 결과 | 기대값과 최종 설정·업무 동작 | 다른 정책·관리 도구·앱 덮어쓰기 |

정책 GUID로 AD 정보와 파일을 대응시킨다. GUID를 다른 정책 이름과 섞거나 문제를 해결하려고 SYSVOL 파일을 수동 복사·삭제하지 않는다. 정책 한 개와 전체 SYSVOL 복구는 영향 범위가 다르다.

## 원본 검증 체크리스트를 클라이언트 증거에 연결한다

원본 구축 가이드의 “정책 적용 확인”은 GPMC 링크의 존재와 클라이언트의 처리 결과를 모두 남기는 것으로 보강한다. 같은 장비에서 비교하더라도 로그인 사용자와 관리자 세션이 다르면 사용자 정책 결과도 달라질 수 있다.

1. 변경 전 보고서·실제 값·이벤트 시각을 보존한다.
2. 실패한 처리의 Activity ID를 찾아 GroupPolicy/Operational 로그의 시작과 종료를 묶는다.
3. 오류가 가리키는 DNS·SYSVOL·권한·확장 중 해당 의존성만 조사한다.
4. 승인된 새로고침·재로그온·재시작 이후 새 Activity ID와 결과를 다시 수집한다.
5. 대상·비대상 장비의 실제 값과 업무 기능을 비교한다.

새로고침하면 Activity ID도 바뀌므로 이전 로그 묶음을 새 결과로 오인하지 않는다. 명령 성공 문구와 실제 적용은 구분한다. [GPO 처리 인스턴스와 Activity ID](https://learn.microsoft.com/en-us/troubleshoot/windows-server/group-policy/applying-group-policy-troubleshooting-guidance#determine-the-instance-of-group-policy-processing)

변경 이후 생긴 장애라면 GPO 전체 삭제보다 변경한 설정·링크·필터·권한을 기록과 비교한다. 원복 후에도 설정이 남는 항목은 해당 정책의 동작에 맞춰 복구하고 동일한 클라이언트·사용자·업무 조건으로 재검증한다. 원인과 관련 없는 정책을 수정하거나 전체 도메인에 강제 새로고침을 확대하지 않는다.

## 보고서 자체도 내부 정보다

gpresult와 이벤트 로그에는 도메인·계정·서버·정책 이름이 포함될 수 있다. 외부에 전달할 때는 필요한 항목만 추리고 식별 정보를 가린다. 진단에서는 원본을 보관하되, 블로그에는 그대로 올리지 않는다.

---

적용 기준: Windows Server AD DS와 도메인 멤버 Windows의 GPO. 명령 예제는 명령 프롬프트.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.

[연재 목차]({% post_url 2026-08-13-ad-series-guide %})
