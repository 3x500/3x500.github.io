---
layout: single
title: "AD 13. GPO가 적용되지 않을 때 확인할 것"
categories: [Active Directory]
tags: [AD, GPO, gpresult, Troubleshooting]
toc: true
published: false
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

## 보고서 자체도 내부 정보다

gpresult와 이벤트 로그에는 도메인·계정·서버·정책 이름이 포함될 수 있다. 외부에 전달할 때는 필요한 항목만 추리고 식별 정보를 가린다. 진단에서는 원본을 보관하되, 블로그에는 그대로 올리지 않는다.

---

적용 기준: Windows Server AD DS와 도메인 멤버 Windows의 GPO. 명령 예제는 명령 프롬프트.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.
