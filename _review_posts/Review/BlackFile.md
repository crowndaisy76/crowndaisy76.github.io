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

이러한 Vishing 통화는 실시간 중간자 공격({% include wiki_link.html text="AiTM" url="/review/Detailed/AiTM" %}, Adversary-in-the-Middle)과 동기화되어 진행된다. 직원이 사내 시스템과 유사한 위장 SSO 페이지에 자격 증명을 입력하면, 공격자는 이를 정식 SSO로 즉각 릴레이한다. 이후 시스템이 2차 인증(MFA)을 요구할 때, 직원이 이를 정상적인 보안 업데이트 과정으로 오인하여 인증을 승인하도록 유도함으로써 기존의 MFA 방어망을 무력화한다.

## 2. Persistence & Exfiltration
접근 권한을 획득한 공격자는 향후 지속적인 접근을 보장하기 위해 새로운 MFA 디바이스를 공격자 통제하에 등록한다. 이후 SaaS 환경(SharePoint, OneDrive 등)으로 수평 이동하여 본격적인 데이터 탈취를 시작한다.

주목할 점은 자동화된 스크립트와 API의 적극적인 활용이다. 공격자는 Microsoft Graph API, Python(requests 라이브러리), PowerShell 등을 동원하여 데이터를 탈취했으며, 탈취한 유효 세션 쿠키를 재사용해 제어 인프라로 데이터를 직접 스트리밍했다.

이 과정에서 정교한 탐지 우회 기법이 관찰되었다.

- 로그 우회: 직접 다운로드(Download) 명령 대신 단순 웹 조회(Fetch)로 위장하여, SOC 환경에서 위험도가 낮게 평가되는 FileAccessed 이벤트로 로그를 남겼다.

- ClientAppId 위장: 조건부 접근 제어를 우회하기 위해 접속 앱 식별자를 'Microsoft Office'로 위장했다. 하지만 Microsoft 365 UAL(Unified Audit Log) 분석 결과, 앱 이름과 달리 실제 UserAgent는 python-requests/2.28.1로 남는 등 스크립팅 엔진을 사용한 흔적(Mismatch)이 식별되었다.

## 3. Extortion
데이터 확보를 마친 공격자는 탈취한 정보를 BlackFile DLS(Data Leak Site)에 게시하겠다고 협박한다. 초기 랜섬 노트는 자동 생성된 일반 Gmail 계정을 통해 발송되며, 내부에는 협박 협상을 위한 익명 메신저(Tox 또는 Session) ID가 포함되어 있다. 만약 피해자가 협상에 응하지 않을 경우, 공격자는 수십 개의 임의 메일 계정을 동원해 사내 직원들에게 스팸성 협박 메일을 쏟아내거나 임원진에게 음성 메시지를 남기는 등 공격적인 압박 전술로 전환한다.

# [MITRE ATT&CK Mapping]
## Initial Access
* Vishing - [T1566.004](https://attack.mitre.org/techniques/T1566/004/)
* Impersonation - [T1684.001](https://attack.mitre.org/techniques/T1684/001/)

## Credential Access & Defense Evasion
* Adversary-in-the-Middle - [T1557](https://attack.mitre.org/techniques/T1557/)
* Masquerading - [T1036](https://attack.mitre.org/techniques/T1036/)

## Persistence
* Account Manipulation: Device Registration - [T1098.005](https://attack.mitre.org/techniques/T1098/005/)
* Valid Accounts: Cloud Accounts - [T1078.004](https://attack.mitre.org/techniques/T1078/004/)

## Collection
* Data from Information Repositories: SharePoint - [T1213.002](https://attack.mitre.org/techniques/T1213/002/)

## Exfiltration
* Automated Exfiltration - [T1020](https://attack.mitre.org/techniques/T1020/)

## Impact
* Financial Theft - [T1657](https://attack.mitre.org/techniques/T1657/)

# [Defense Strategies]

# [Referenced]
* [Google Threat Intelligence Group](https://cloud.google.com/blog/topics/threat-intelligence/blackfile-vishing-extortion-operation/?hl=en)