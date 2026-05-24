---
layout: default
title: UNC6671 - BlackFile
date: 2026-05-24
---

# [Executive Summary]
본 사건은 위협 그룹 UNC6671이 Vishing과 AiTM 기법을 악용하여 기업의 SSO 계정을 탈취하고, 클라우드 데이터를 유출하여 금전을 갈취한 침해 사고이다. 사회공학적 기법으로 초기 침투에 성공한 뒤, 자동화 스크립트로 데이터를 선별 확보하고 BlackFile DLS에 게시하여 협박하는 공격 방식이 핵심이다.
# [Technical Analysis]
## 1. Initial Access & Credential Access
UNC6671은 보안 도구의 탐지를 피하고자 사내 직원의 개인 휴대전화로 직접 전화를 거는 {% include wiki_link.html text="Vishing" url="/review/Detailed/Vishing" %} 기법을 사용했다. 이들은 사내 IT 헬프데스크로 위장하여 '패스키(Passkey) 마이그레이션'이나 'MFA 업데이트'가 필요하다는 명분으로 타깃을 속였다. 과거에는 조직 맞춤형 도메인을 썼으나, 최근에는 Tucows에 등록된 서브도메인(예: enrollms, passkeyms) 기반 모델로 전환하여 신뢰도를 높였다.

이러한 Vishing 통화는 실시간 중간자 공격({% include wiki_link.html text="AiTM, Adversary-in-the-Middle" url="/review/Detailed/AiTM" %})과 동기화되어 진행된다. 직원이 사내 시스템과 유사한 위장 SSO 페이지에 자격 증명을 입력하면, 공격자는 이를 정식 SSO로 즉각 릴레이한다. 이후 시스템이 2차 인증(MFA)을 요구할 때, 직원이 이를 정상적인 보안 업데이트 과정으로 오인하여 인증을 승인하도록 유도함으로써 기존의 MFA 방어망을 무력화한다.

## 2. Persistence & Exfiltration
접근 권한을 획득한 공격자는 향후 지속적인 접근을 보장하기 위해 새로운 MFA 디바이스를 공격자 통제하에 등록한다. 이후 SaaS 환경(SharePoint, OneDrive 등)으로 수평 이동하여 본격적인 데이터 탈취를 시작한다.

주목할 점은 자동화된 스크립트와 API의 적극적인 활용이다. 공격자는 Microsoft Graph API, Python(requests 라이브러리), PowerShell 등을 동원하여 데이터를 탈취했으며, 탈취한 유효 세션 쿠키를 재사용해 제어 인프라로 데이터를 직접 스트리밍했다.

이 과정에서 정교한 탐지 우회 기법이 관찰되었다.

- 로그 우회: 직접 다운로드 명령 대신 단순 웹 조회(Fetch)로 위장하여, SOC 환경에서 위험도가 낮게 평가되는 FileAccessed 이벤트로 로그를 남겼다.

- ClientAppId 위장: 조건부 접근 제어를 우회하기 위해 접속 앱 식별자를 'Microsoft Office'로 위장했다. 하지만 {% include wiki_link.html text="Microsoft 365 UAL(Unified Audit Log)" url="/review/Detailed/M365_UAL" %} 분석 결과, 앱 이름과 달리 실제 UserAgent는 python-requests/2.28.1로 남는 등 스크립팅 엔진을 사용한 흔적(Mismatch)이 식별되었다.

## 3. Extortion
데이터 확보를 마친 공격자는 탈취한 정보를 BlackFile {% include wiki_link.html text="DLS(Data Leak Site)" url="/review/Detailed/DLS" %} 에 게시하겠다고 협박한다. 초기 랜섬 노트는 자동 생성된 일반 Gmail 계정을 통해 발송되며, 내부에는 협박 협상을 위한 익명 메신저(Tox 또는 Session) ID가 포함되어 있다. 만약 피해자가 협상에 응하지 않을 경우, 공격자는 수십 개의 임의 메일 계정을 동원해 사내 직원들에게 스팸성 협박 메일을 쏟아내거나 임원진에게 음성 메시지를 남기는 등 공격적인 압박 전술로 전환한다.

# [MITRE ATT&CK Mapping]
## Initial Access
* **Phishing: Spearphishing Voice** - [T1566.004](https://attack.mitre.org/techniques/T1566/004/)
* **Social Engineering: Impersonation** - [T1684.001](https://attack.mitre.org/techniques/T1684/001/)

## Credential Access & Defense Evasion
* **Adversary-in-the-Middle** - [T1557](https://attack.mitre.org/techniques/T1557/)
* **Masquerading** - [T1036](https://attack.mitre.org/techniques/T1036/)

## Persistence
* **Account Manipulation: Device Registration** - [T1098.005](https://attack.mitre.org/techniques/T1098/005/)
* **Valid Accounts: Cloud Accounts** - [T1078.004](https://attack.mitre.org/techniques/T1078/004/)

## Collection
* **Data from Information Repositories: SharePoint** - [T1213.002](https://attack.mitre.org/techniques/T1213/002/)

## Exfiltration
* **Automated Exfiltration** - [T1020](https://attack.mitre.org/techniques/T1020/)

## Impact
* **Financial Theft** - [T1657](https://attack.mitre.org/techniques/T1657/)

# [Defense Strategies]

## **1. 자동화된 선제 방어 및 인증 체계 강화**

* **Phishing-Resistant MFA 도입**
  * SMS 인증이나 단순 푸시 알림 방식의 MFA에서 탈피해야 한다. UNC6671이 구사한 실시간 중간자 공격(AiTM) 및 Vishing 전술을 원천 차단하기 위해, 도메인 검증 기능이 내장된 FIDO2 규격의 하드웨어 보안 키나 패스키(Passkey)로 전환해야 한다.


* **Credential Guarding 도입**
  * 사용자가 가짜 도메인에 기업 계정 패스워드를 입력하는 순간을 실시간으로 포착해야 한다.
  * **Google Workspace:** '비밀번호 알림(Password Alert)' 기능을 활성화하여 사내 패스워드 해시 값이 승인되지 않은 도메인에 입력되는지 모니터링한다.
  * **Microsoft 환경:** 'Microsoft Defender 자격 증명 보호' 및 'SmartScreen'을 활용해 평판이 낮거나 피싱으로 의심되는 사이트에서의 자격 증명 제출을 가로챈다. 사용자 실수로 악성 페이지와 상호작용하더라도 자동 패스워드 초기화나 보안 알림이 트리거되도록 안전장치를 마련한다.

## **2. ID 관리자({% include wiki_link.html text="IdP" url="/review/Detailed/"IdP %}) 및 인프라 로그 감시**

* **IdP 로그 모니터링**
  * 공격자가 침투 후 권한 유지를 위해 새로운 기기를 등록하는 행위를 추적해야 한다.
  * IdP 로그를 분석하여, 2차 인증 실패(`user.authentication.auth_via_mfa failures`) 또는 인증 포기(`Abandoned challenges`) 이벤트가 발생한 직후 **새로운 MFA 수단이 등록(`system.multifactor.factor.setup`)되는 이상 흐름**을 탐지한다.


* **접속 인프라 연계 분석**
  * 사용자의 평소 상주 지역이나 로그인 패턴과 비교했을 때, 일반적인 상용 VPN 주소나 호스팅 제공업체(AWS, Azure 등)의 IP 대역에서 발생하는 비정상적인 인증 시도를 탐지하고 경고를 발생시킨다.


* **미인증 기기의 SDK User-Agent 감시**
  * 해당 사용자 프로필에 등록되거나 연결된 적이 없는 생소한 기기에서 특정 IdP SDK(소프트웨어 개발 키트) 관련 User-Agent를 통해 접속을 시도하는지 모니터링한다.

## **3. SaaS API 및 데이터 유출 모니터링**

* **SaaS API 활동 감사**
  * Microsoft 365, SharePoint, Salesforce 등의 감사 로그를 상시 모니터링한다. 일반적인 웹 브라우저가 아닌 PowerShell, Python 등의 스크립팅 엔진 식별자가 포함된 User-Agent를 통해 단시간에 대량의 파일 다운로드(`FileDownloaded` 또는 `FileAccessed`)가 발생하는지 추적한다.


* **`FileAccessed` 위험도 재평가**
  * 보안관제센터(SOC)는 공격자가 프로그래밍 라이브러리(Python, Go 등)나 CLI(명령줄 도구)를 사용해 접근한 경우, 파일을 직접 다운로드한 이벤트(`FileDownloaded`)뿐만 아니라 **단순 조회 이벤트(`FileAccessed`)도 동일한 심각도(High/Critical)로 취급하여 조사**해야 한다.


* **Direct File Streaming 탐지**
  * 감사 로그 중 `AppAccessContext` 필드에 헤드리스 클라이언트(화면이 없는 자동화 스크립트 등)가 찍히거나, 인간의 브라우징 속도로는 불가능할 만큼 짧은 시간 동안 방대한 양의 파일에 접근(`FileAccessed`)한 비정상적 패턴을 정밀 감사한다.

# [Referenced]
* [Google Threat Intelligence Group](https://cloud.google.com/blog/topics/threat-intelligence/blackfile-vishing-extortion-operation/?hl=en)