---
layout: single
title: "AD 15. Site·Subnet과 DC 선택"
categories: [Active Directory]
tags: [AD, Site, Subnet, DC Locator]
toc: true
published: false
---

같은 도메인을 사용하는데 지점 PC가 먼 지역의 DC와 통신할 수 있다. OU가 지점별로 나뉘었다고 가까운 DC가 자동으로 선택되는 것은 아니다. 객체 관리 구조와 네트워크 위치를 구분해야 한다.

Domain·OU가 업무 분류라면 Site·Subnet은 네트워크 지도에 가깝다. AD의 Subnet 객체를 Site와 연결해 클라이언트 위치 판단에 활용한다. Site Link는 사이트 사이 복제 경로와 비용·일정 등을 표현한다. [Microsoft Site·Subnet·Site Link](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/replication/active-directory-replication-concepts)

## 지점 이름만 만드는 것으로 끝나지 않는다

| 항목 | 확인할 내용 |
|---|---|
| Site | 네트워크 위치를 어떤 단위로 정의했는가? |
| Subnet 매핑 | 실제 클라이언트 주소 범위가 올바른 Site에 연결됐는가? |
| DC 배치 | 해당 Site에서 사용할 수 있는 DC가 있는가? |
| Site Link | 회선과 복제 비용·일정을 어떻게 반영했는가? |

새 지점이나 VPN 주소 대역을 추가했는데 Subnet 등록을 놓치면 위치 판단이 기대와 달라질 수 있다. 지점별 OU를 추가하는 작업과 별도 확인 항목으로 관리한다.

## 클라이언트 관점에서 조회한다

해당 지점의 도메인 멤버 PC에서 실행하는 진단 예제다. 실행 계정·도구 설치 상태에 따라 권한과 사용 가능 여부를 확인한다.

```text
nltest /dsgetsite
nltest /dsgetdc:ad.example.com
```

첫 결과에서 클라이언트 Site를, 두 번째에서 검색된 DC와 Site 관련 정보를 확인한다. 이는 DC 검색 결과이며, 해당 PC의 모든 애플리케이션이 그 DC를 사용한다는 의미는 아니다. 애플리케이션은 서버를 직접 지정하거나 연결을 캐시할 수도 있다. [Microsoft DC Locator](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/dc-locator), [Microsoft nltest](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/cc731935(v=ws.11))

## 가까운 DC를 못 쓰는 원인

로컬 DC의 상태·기능, 사이트별 DNS 레코드, 서브넷 매핑, 회선·방화벽을 함께 본다. 가까운 DC가 없거나 사용할 수 없으면 다른 위치의 DC를 선택할 수 있다. “항상 같은 지점 DC만 사용해야 정상”이라고 판단하지 않는다.

변경 후에는 관리 콘솔의 지도뿐 아니라 실제 지점 PC에서 검색 결과와 인증 지연을 확인한다. 관리자가 본사 PC에서 확인한 결과로 지점 동작을 대신할 수는 없다.

---

적용 기준: Windows Server AD DS의 Site·Subnet·DC Locator. 버전별 DC 검색 차이는 공식 문서 참조.  
공식 근거 검토일: 2026-11-10. 명령어 예제는 이 글 작성을 위해 실제 AD에서 실행하지 않았다.
