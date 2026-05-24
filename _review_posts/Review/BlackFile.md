---
layout: default
title: BlackFile by UNC6671
date: 2026-05-24
---
Extortion campaign by UNC6671(BlackFile)
voice phising (vishing) 과 single sign-on(SSO)으로 공격
adversary-in-the-middle (AiTM) 기법으로 기존 경계 방어와 MFA를 우회 -> 클라우드 환경의 접근 권한 획득
Microsoft 365와 Okta 인프라를 주 타겟 <- Python, Powershell 스크립트

2026년 초 등장
북미, 호주, 영국을 주 타겟
별도의 BlackFile 데이터 유출 사이트(DLS)를 구축
취약점보다는 사회공학에 집중

피싱을 통해 대부분의 초기 침투
보안 도구 우회를 위해 개인 휴대전화로 직접 전화
기존의 조직 맞춤형 자격 증명 탈취 도메인 -> 서브도메인 기반 모델(Tucows)
위장을 위해 "패스키", "등록"과 같은 주제를 명시적으로 언급하는 서브도메인 사용
- `<organization>.enrollms[.]com`
- `<organization>.passkeyms[.]com`
- `<organization>.setupsso[.]com`

피싱으로 live adversary-in-the-middle (AitM) attack 수행
- Redirection: 유사한 SSO 서브도메인으로
- Credential Capture: 이름,비밀번호 입력하면 바로 정식 SSO로 전송
- MFA Bypass: 공격자에게 인증 코드 전송
- Device Registration: 접근 권한 획득 시 새로운 MFA 장치 등록하여 지속성 확보

SaaS 횡적 이동을 통해 데이터 탈취
Microsoft Graph, python-request, Powershell 등을 활용하여 SharePoint, OneDrive 탈취
초기 침투에서 획득한 쿠키(FedAuth)를 통해 제어 인프라로 직접 스트리밍
-> 탐지 우회
FileDownloaded보다 FileAccesssed 사용


Microsoft 365 Unified Audit Log (UAL) 분석
-> Microsoft Office의 ClientAppID를 위조했는데 UserAgent가 python-requests/2.28.1 또는 WindowsPowerShell/5.1과 같은 스크립팅 엔진

```
{
  "CreationTime": "2026-02-24T14:36:15",
  "Operation": "FileDownloaded",
  "Workload": "SharePoint",
  "ClientIP": "179.43.185.226", 
  "UserId": "victim.user@organization.com",
  "UserAgent": "python-requests/2.28.1",
  "ApplicationDisplayName": "Microsoft Office",
  "IsManagedDevice": false,
  "SourceFileName": "2382_REDACTED_MSA_v3.docx",
  "SourceRelativeUrl": "Shared Documents/Legal/MasterMSA/Archive",
  "SiteUrl": "https://organization.sharepoint.com/sites/Legal_Archive/",
  "AppAccessContext": {
    "ClientAppId": "d3590ed6-52b3-4102-aeff-aad2292ab01c",
    "ClientAppName": "Microsoft Office",
    "TokenIssuedAtTime": "1601-01-01T00:00:00"
  }
}
```
Figure 1: FileDownloaded event observed in early UNC6671 intrusions

```
{
  "CreationTime": "2026-03-18T20:06:41",
  "Operation": "FileAccessed",
  "Workload": "SharePoint",
  "UserId": "victim.user@company.com",
  "ClientIP": "179.43.185.226", 
  "UserAgent": "python-requests/2.28.1",
  "ApplicationDisplayName": "python-requests",
  "IsManagedDevice": false,
  "SourceRelativeUrl": "Shared Documents/Data Analytics/Power BI Version History",
  "SourceFileName": "Weekly Production Report.pbix",
  "SiteUrl": "https://company.sharepoint.com/sites/ProductionOps/",
  "AppAccessContext": {
    "ClientAppName": "python-requests",
    "CorrelationId": "b94b01a2-2019-c000-2262-5ff1d0ff6cc8"
  }
}
```
Figure 2: FileAccessed event from later UNC6671 intrusions


랜섬노트 -> 연락 -> BlackFile 브랜드
거부 시 공격적 협박(스팸 메일 대량 발송, 최고 경영진 협박 음성 메시지, Swatting)
```
**Subject:** [COMPANY NAME] DATA BREACH 72 HOURS TO CONTACT US**From:** `[pseudorandom_alphanumeric_string]@gmail.com`

Hello [Company Name] Executives and HR,

We have managed to export ~[X] TB of data from your network due to your terrible security practices and negligent data storing practices.

Here is a brief overview of data exported from your network:

1. [X]+ GB of internal company files (SharePoint & OneDrive) containing confidential business processes, NDAs, project cost estimates, subcontractor contracts, and HR records.
    
2. Tens of thousands of emails from executive mailboxes, including confidential documents.
    
3. Complete CRM and support ticket exports (Salesforce & Zendesk) containing hundreds of thousands of customer records, PII, billing details, and communication logs.
    
4. Complete corporate directory (Entra) dumps including employee names, mobile numbers, job titles, and hierarchy.
    
5. ~[X] ServiceNow IT infrastructure records (computers, servers, cloud resources).
    

You have exactly 72 hours to contact the [Tox / Session] ID provided below. If you fail to contact the ID provided by us within the timeframe stated, we will be forced to publish your data to the public. We will also be forced to contact each company you work with via the employee team contact phone numbers and email addresses provided and explain how [Company Name] has terrible security protocols and does not care about its customers.

We are willing to engage in good faith negotiation terms. Upon contacting us, a full list of all data exported from your network will be sent to you for review. You will be able to pick up to 3 files to confirm and verify we have what we are claiming.

**[Tox / Session] ID:** [Unique Alphanumeric String]

Silence may not always be wise in situations like this. We will not be ignored. Make the right choice and cooperate with us so this can be a learning experience for you.
```
Figure 3: Generalized example initial unbranded extortion note from UNC6671

```
**Subject:** [COMPANY NAME] DATA BREACH 72 HOURS TO CONTACT US**From:** `[pseudorandom_alphanumeric_string]@gmail.com`

Dearest executive,

You have picked to ignore the first deadline to contact us. That is not smart do not ignore us it will only make things worse. We are BlackFile. Do not play games with us. We are giving a final deadline of 72 hours to contact us so we can reach an agreement.

We copied over [X] TB+ of data from your SharePoint & M365 instance (legal documents, operational documents, client documents, sales documents, development documents, etc) over [X]gb of Salesforce data, full ZenDesk support ticket export for [X]+ customers, ALL ticket history including old and new tickets and their contents. Total taken from your network is over [X]TB+

Do not be alarmed as you can secure the proteciton of your data by choosing to work with us. Nothing taken from your network has been disclosed to the public or shared with third parties as of now.

Reach out to us on session to receive all details and evidense that we accessed your network. We will use Session to communicate with you. You can get Session by visiting getsession(.)org

Reach out to the following ID using Session: **[Unique Session ID]**

Do not reply to this email. Instead alert the rest of your HR and SOC/IT Security Team. We give you a final deadline of 72 hours to confirm reciept that you received this email by contacting us on Session.

If you fail to contact us a second time then a majority of the emails taken from your network will receive a notification from us explaining you failed to come to an agreement with us to protect your customers PII and other sensitive information. Additionally we will message journalists about this breach and your failure to come to a resolution with us before finally uploading all data taken from you to our blog for the public.

Do not let a data recovery company tell you not to negotate us we are BlackFile and we do not play games. The data we took from you can seriously damage your reputation if released is it really worth having that happen over ignoring us?

Blackfile
```
Figure 4: Generalized example follow up extortion email which included branding not present in initial messages

랜섬노트의 진화
2026 초기: 공격적, 짧은 기한(24/48시간)
1월말: 72시간, 표준화(`[COMPANY NAME] DATA BREACH 72 HOURS TO CONTACT US`)

초기: 이름 X, Tox (a peer-to-peer instant messaging protocol), 외부 이메일
2026년 2월: BlackFile, Session (a decentralized, privacy-focused messenger), 내부 이메일

2026년 2월 BlackFile Data Leak Site (DLS)
광고/색인화 X, 제한된 샘플만 유출
![[Pasted image 20260524193327.png]]
![[Pasted image 20260524193343.png]]

2026년 4월 사이트 종료
2026년 5월 11일 메시지 보여주려고 잠깐 돌아옴 ("BlackFile is shutting down… under this name.")

![[Pasted image 20260524193445.png]]

막기
- Deploy Credential Guarding: 암호 입력 시 도메인 승인 확인
- Implement Phishing-Resistant MFA: FIDO2 구현
- Monitor IdP Logs: 인증 실패 시 공급자 로그에서 검토
- Correlate Infrastructure: 비정상적에서 인증 시도 경고
- Audit SaaS API Activity: 비정상적인 양의 파일 다운로드 감시
- Monitor User-Agents: 특정 사용자 에이전트 모니터링
- Re-Evaluate "Access" Severity: FileAccessed 중요
- Audit for Direct File Streaming: FileAccessed

---
# [Executive Summary]
본 사건은 위협 그룹 UNC6671이 Vishing과 AiTM 기법을 악용하여 기업의 SSO 계정을 탈취하고, 클라우드 데이터를 유출하여 금전을 갈취한 침해 사고이다. 사회공학적 기법으로 초기 침투에 성공한 뒤, 자동화 스크립트로 데이터를 선별 확보하고 BlackFile DLS에 게시하여 협박하는 공격 방식이 핵심이다.
# [Technical Analysis]
{% include wiki_link.html text="Test " url="/review/Detailed/Test.md" %}

# [MITRE ATT&CK Mapping]

# [Defense Strategies]
