---
layout: default
title: Microsoft 365 UAL
date: 2026-05-24
---

# [Microsoft 365 UAL (Unified Audit Log)]

Microsoft 365 UAL은 SharePoint Online, OneDrive for Business, Exchange Online, Microsoft Teams, Azure Active Directory(Entra ID) 등 Microsoft 365 SaaS 생태계 전반에서 발생하는 모든 사용자 및 관리자의 활동 사건을 한곳에 모아 제공하는 중앙 집중식 로그 저장소이다.

클라우드 기반의 엔터프라이즈 환경을 노리는 위협이 급증함에 따라, 침해 사고 조사 시 비침해성 감사 데이터를 확보할 수 있는 핵심 포렌식 아티팩트로 자리 잡고 있다.

## **1. 포렌식적 가치 및 중요성**

클라우드 침해 사고 분석 시 UAL이 가지는 핵심적인 포렌식 가치는 다음과 같다.

* **가시성의 통합 (Unified Visibility):** 과거에는 메일 로그(Exchange), 파일 접근 로그(SharePoint)를 각각 독립된 자산에서 수집해야 했으나, UAL은 단일 타임라인 상에서 사내 직원의 클라우드 활동 전체를 연계 분석할 수 있도록 지원한다.
* **비정상적 수평 이동(Lateral Movement) 식별:** 공격자가 피싱 등으로 최초 침투 계정을 확보한 뒤, 사내 Teams 메신저를 통해 다른 직원에게 악성 링크를 던지거나 SharePoint 내부망을 뒤져 민감 문서를 탐색하는 흐름을 완벽하게 추적할 수 있다.
* **비동기 데이터 유출(Exfiltration) 흔적 추적:** 대량의 파일 다운로드나 외부 공유 링크 생성 행위 등을 포착하여 기업의 지적재산권(IP) 유출 규모를 산정하는 결정적 증거가 된다.

## **2. 핵심 분석 이벤트**

UAL 분석 시 침해 사고의 단서를 제공하는 주요 오퍼레이션과 필드는 다음과 같다.

* **`FileDownloaded` 및 `FileAccessed` (SharePoint / OneDrive)**
    
    사용자가 파일을 로컬로 다운로드했거나 웹상에서 열람한 행위를 기록한다. 공격자가 대량의 문서를 탈취할 때 발생하는 대량의 `FileDownloaded` 이벤트를 정밀 분석하여 유출 범위를 특정한다.


* **`MailItemsAccessed` (Exchange Online)**
    
    특정 메일 상자(Mailbox)의 메시지를 읽었을 때 기록되는 고가치 이벤트이다. 비즈니스 이메일 침해(BEC) 공격자가 타깃의 메일 본문이나 첨부파일을 훔쳐봤는지 여부를 입증하는 직접적인 증거로 쓰인다.


* **`UserLoggedIn` 및 `UserLoginFailed` (Entra ID)**
    
    로그인 성공 및 실패 이벤트를 기록한다. 다중 인증(MFA) 우회 기법(AiTM 등)이 작동한 시점의 소스 IP, 자격 증명 유형, 로그인 결과를 파악하여 최초 침투 경로를 재구성할 수 있다.


* **`UserAgent` 및 `ClientIP` 필드**

    각 이벤트가 발생한 브라우저 환경과 IP 주소를 제공한다. 일반적인 웹 브라우저가 아닌 `python-requests`나 `WindowsPowerShell` 같은 스크립팅 엔진 식별자가 `UserAgent`에 남은 경우, 공격자가 자동화 도구를 사용해 클라우드 데이터를 대량으로 긁어갔음을 입증하는 강력한 지표가 된다.
