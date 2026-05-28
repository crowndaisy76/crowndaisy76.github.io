---
layout: default
title: "Undermining the Trust Boundary: A Stealthy Third-Party Intrusion"
date: 2026-05-28
---

# [Executive Summary]

본 사건은 신뢰 관계가 구축된 외부 파트너사의 자산 침해로부터 시작된 고도화된 공급망 및 내부망 침투 사례이다.

공격자는 탐지 시그니처가 발생하는 전형적인 악성코드나 취약점을 사용하는 대신, 타깃 조직이 이미 환경 내에서 정식 서명하고 신뢰하던 엔터프라이즈 관리 도구인 {% include wiki_link.html text="HPE Operations Agent(HPE OA)" url="/review/Detailed/OA" %}를 전면에 악용하였다. 이를 통해 공격자는 정상적인 관리자 업무 흐름으로 위장하여 초기 보안 탐지망을 완벽히 우회하였다.

이후 도메인 인프라를 장악한 공격자는 Windows 인증 프로세스의 아키텍처 구조를 악용하여 악성 {% include wiki_link.html text="네트워크 프로바이더" url="/review/Detailed/Network_Provider" %} 및 {% include wiki_link.html text="패스워드 필터" url="/review/Detailed/Password_Filter" %}를 등록하는 정교한 자격 증명 가로채기 기법을 전개하였다. 이를 통해 일반 텍스트 형태의 사용자 자격 증명과 패스워드 변경 이벤트를 실시간으로 수집하였으며, 확보한 고권한 계정을 교두보 삼아 내부 핵심 인프라 전반에 걸쳐 은밀한 횡적 이동과 장기적인 지속성을 확립하였다.


# [Technical Analysis]

## 1. Initial Access

공격자는 외부망에 노출된 웹 서버 2곳에서 최초 거점을 확보하고 Errors.aspx라는 웹쉘을 배포하였다. 해당 웹 서버에서 취약점 공격이 수행된 흔적은 발견되지 않았으나, 공격자가 확보한 웹쉘을 통해 추가적인 스크립트 실행 및 바이너리 배포가 이루어졌다.

이후 외부 악성 도메인과 통신하고 있던 내부 워크스테이션을 추적하는 과정에서, 외부 파트너사가 위탁 관리하던 중앙 집중식 관리 콘솔인 HPOM 인프라가 장악된 사실이 확인되었다. 공격자는 이 제어 권한을 남용하여 각 호스트에 설치된 HPE OA를 통해 악성 VBScript인 abc003.vbs를 다수의 서버와 도메인 컨트롤러에 정상 명령처럼 하향식으로 배포하였다.

배포된 abc003.vbs는 시스템 내부 네트워크 구성 및 액티브 디렉터리 구조를 정찰하였으며, 내부 탐지망을 우회하기 위해 파워쉘을 호출하여 외부 IP 주소를 식별하는 초기 정보 수집 행위를 수행하였다.

## 2. Credential Access & Exfiltration

공격자는 최초 침투 성공 후, 도메인 내 고권한 계정을 지속적으로 확보하기 위해 {% include wiki_link.html text="도메인 컨트롤러" url="/review/Detailed/Domain_Controller" %}를 대상으로 윈도우의 내부 인증 프로세스 자체를 변조하는 고도화된 자격 증명 가로채기 전술을 전개하였다.

### 네트워크 프로바이더(Network Provider) 하이재킹

공격자는 도메인 컨트롤러 DC01에 mslogon이라는 이름의 정상적인 네트워크 프로바이더를 위장 등록하여 하위 인증 체계를 장악하였다. 윈도우 인증 메커니즘의 특성상 등록된 프로바이더는 사용자 인증 정보를 가로챌 수 있는 권한을 가진다. 배포된 mslogon.dll은 윈도우 자격 증명 관리자 API인 `NPLogonNotify`와 `NPPasswordChangeNotify`를 악용하는데, 이 API들은 시스템에 인증 이벤트가 발생할 때 프로바이더에게 알림을 보내도록 설계된 정상 메커니즘이다. 사용자가 대화형 로그인을 수행하면 `NPLogonNotify`가 트리거되어 입력된 사용자 이름과 패스워드를 평문 형태로 포획하며, 사용자가 패스워드를 변경할 때는 `NPPasswordChangeNotify`가 호출되어 기존 패스워드와 신규 패스워드 쌍을 동시에 가로챈다. 가로챈 평문 자격 증명은 정상적인 미디어 자산처럼 위장하기 위해 C:\Users\Public\Music\abc123c.d 경로에 은밀히 저장되었으며, 공격자는 이를 횡적 이동을 위한 고권한 계정 재사용에 활용하였다.

### LSA 알림 패키지를 통한 악성 패스워드 필터 주입

공격 후반기에는 도메인 컨트롤러 DC01 및 DC02를 대상으로 로컬 보안 인증 알림 패키지 설정에 passms라는 악성 패스워드 필터를 추가 등록하여 감시망을 공고히 했다. 패스워드 필터는 도메인 컨트롤러의 핵심 인증 프로세스인 LSASS에 직접 로드되어 동작한다. 시스템에서 패스워드 설정 및 변경 요청이 발생하면, LSASS는 알림 패키지 레지스트리에 등록된 DLL들의 `PasswordFilter()` API를 호출하는데, 이 함수는 입력 파라미터로 사용자 이름과 패스워드를 가공되지 않은 평문 상태로 전달받는다. 탈취된 데이터는 C:\ProgramData\WindowsUpdateService\UpdateDir\Ipd 파일에 기록되었으며, 이전의 평문 저장 방식과 달리 탐지를 우회하기 위해 Base64 인코딩을 거친 후 DLL 내부에 구현된 독자적인 커스텀 알고리즘으로 이중 암호화하여 저장한다.

### 유출 모듈(msupdate.dll)과의 연계

공격자는 패스워드 필터와 상호작용하며 데이터를 외부로 유출하는 별도의 모듈인 msupdate.dll을 DC01과 DC02에 생성하고 파워쉘을 통해 실행하였다. 실행 시에는 다음 명령어를 사용하였다.

```powershell
start powershell.exe -c "[System.Reflection.Assembly]::LoadFrom('C:\Windows\System32\Com\msupdate.dll'); [WindowsHook.Program]::Main('msupdate')"
```

해당 모듈은 이중 암호화된 Ipd 파일의 내용을 읽어 들인 뒤 두 가지 경로로 유출을 시도하였다. 내부망에서는 SMB 프로토콜을 통해 원격 공유 폴더로 데이터를 전송하며 이미지 파일인 icon02.jpeg로 확장자를 위장하여 적재했다. 동시에 내부 환경 설정 파일에서 추출한 자격 증명을 이용해 지정된 SMTP 서버로 "Update Service"라는 제목의 이메일을 발송하는 방식으로 자격 증명을 외부로 최종 유출하였다.

## 3. Execution & Persistence

공격자는 이미 환경 내에 존재하는 정상적인 엔터프라이즈 자동화 채널을 악용하여 추가적인 악성 스크립트와 파워쉘 명령을 실행하는 방식을 사용했다.

### 신뢰 기반 자동화 채널 및 웹쉘을 통한 명령 실행

초기 정찰 단계에서 공격자는 파트너사가 위탁 관리하던 HPE OA 인프라를 통해 악성 VBScript인 abc003.vbs를 실행하여 내부 네트워크 및 액티브 디렉터리 구조를 파악하였다. 동시에 외부망에 노출된 웹 서버에서는 최초로 심어둔 Errors.aspx 웹쉘과 추가로 변조한 Signoff.aspx 웹쉘을 거점으로 삼아 파워쉘 스크립트를 호출하고, 추가적인 바이너리 배포 및 자격 증명 접근을 위한 후속 작업을 은밀히 트리거하였다.

### 웹쉘의 단계적 확장과 영구 지속성 확보

공격자는 초기 거점인 Errors.aspx 웹쉘을 이용해 디스크에 새로운 파일을 작성한 뒤, 기존의 정상적인 Signoff.aspx 코드를 악의적으로 수정하여 탐지를 회피하였다. 이후 윈도우 임시 디렉터리 경로에서 ghost.inc라는 추가 웹쉘을 실행하였는데, 이 파일은 임의의 시스템 명령 실행뿐만 아니라 파일의 업로드와 다운로드 기능을 제어하는 핵심적인 지속성 거점 역할을 수행하였다. 이에 더해 앞서 도메인 컨트롤러 DC01과 DC02에 등록했던 악성 네트워크 프로바이더 및 패스워드 필터와 같은 하위 인증 구성 요소들이 시스템 재시작 후에도 자동으로 로드되도록 설정함으로써 강력한 지속성 체계를 완성하였다.

### ngrok 터널링을 이용한 방화벽 우회 및 원격 제어

공격자는 내부망 정찰 결과를 바탕으로 시스템 내부에서 외부 인터넷으로 나가는 아웃바운드 연결이 허용된 특정 서버들을 식별하였다. 엄격한 외부 경계 방화벽 통제를 우회하기 위해 공격자는 해당 서버들에 ngrok 도구를 배포하고 실행하여 공격자의 외부 인프라와 연결되는 암호화된 터널을 생성하였다. 이 터널을 통해 외부 방화벽 포트를 추가로 개방하지 않고도 내부 시스템에 대한 원격 데스크톱(RDP) 세션을 안정적으로 수립하였으며, 이를 발판 삼아 핵심 자산으로의 추가적인 횡적 이동과 지속적인 원격 접근 채널을 유지하였다.

## 4. Lateral Movement
공격자는 획득한 고권한 계정과 암호화된 터널링 인프라를 결합하여 내부망의 핵심 자산들을 차례로 장악하는 횡적 이동을 전개하였다.

### 고권한 계정과 원격 데스크톱을 통한 중요 자산 침투

탈취한 도메인 고권한 계정을 확보한 공격자는 환경 전체를 대상으로 원격 데스크톱 세션을 공격적으로 수립하였다. 이를 통해 내부망 깊숙이 위치한 SQL 서버인 SQL-01과 기업의 브레인 역할을 하는 도메인 컨트롤러 등 실질적인 데이터와 권한이 집중된 중요 장치들에 직접 액세스하며 장악력을 넓혀나갔다.

### ngrok 터널링 기반의 공격 출처 은닉

네트워크 경계 기반의 모니터링을 무력화하고 자신의 실제 공격 인프라 IP를 숨기기 위해 공격자는 앞서 구축한 ngrok 터널을 적극적으로 활용하였다. 분석 결과 공격자가 수행한 원격 데스크톱 연결의 상당수가 내부 서버인 SQL-01에 호스팅되어 있던 ngrok 터널 내부에서 시작된 것으로 확인되었다. 이는 보안 관제 시스템에서 보기에는 단순한 내부 서버 간의 정상적인 RDP 통신처럼 보이게 만들어, 외부 공격자의 가시성을 완전히 차단하고 탐지 분석을 까다롭게 만들었다.

### WMI 기반의 원격 실행을 통한 터널링 인프라 확장

공격자는 이미 장악한 웹 서버를 발판 삼아 내부망의 다른 장치들로 ngrok 통제권을 확장하기 위해 윈도우 관리 도구(WMI)를 원격 명령 실행 수단으로 오용하였다. WMI 원격 실행 기능을 통해 내부의 추가적인 장치들에 ngrok 도구를 하향식으로 배포하고 구동시켰으며, 이로 인해 내부망 전역에 다중 암호화 아웃바운드 터널이 형성되어 공격자가 방화벽 제약 없이 자유롭게 내부망을 활보할 수 있는 환경이 조성되었다.

# [MITRE ATT&CK Mapping]

## Initial Access

* **Trusted Relationship** - [T1199](https://attack.mitre.org/techniques/T1199/)

## Execution

* **Command and Scripting Interpreter: PowerShell** - [T1059.001](https://attack.mitre.org/techniques/T1059/001/)
* **Command and Scripting Interpreter: Visual Basic** - [T1059.005](https://attack.mitre.org/techniques/T1059/005/)
* **Windows Management Instrumentation** - [T1047](https://attack.mitre.org/techniques/T1047/)

## Persistence

* **Modify Registry** - [T1112](https://attack.mitre.org/techniques/T1112/)
* **Server Software Component: Web Shell** - [T1505.003](https://attack.mitre.org/techniques/T1505/003/)
* **Modify Authentication Process: Password Filter** - [T1556.002](https://attack.mitre.org/techniques/T1556/002/)

## Credential Access

* **Modify Authentication Process** - [T1556](https://attack.mitre.org/techniques/T1556/)
* **Credentials from Password Stores** - [T1555](https://attack.mitre.org/techniques/T1555/)

## Discovery

* **System Network Configuration Discovery** - [T1016](https://attack.mitre.org/techniques/T1016/)
* **Remote System Discovery / Account Discovery** - [T1018](https://attack.mitre.org/techniques/T1018/) / [T1087](https://attack.mitre.org/techniques/T1087/)

## Lateral Movement

* **Remote Services: Remote Desktop Protocol** - [T1021.001](https://attack.mitre.org/techniques/T1021/001/)

## Command and Control

* **Protocol Tunneling** - [T1572](https://attack.mitre.org/techniques/T1572/)

## Exfiltration

* **Exfiltration Over Alternative Protocol** - [T1048](https://attack.mitre.org/techniques/T1048/)

# [Defense Strategies]

## 엔드포인트 보호 및 가시성 강화

급변하는 공격 도구와 기법에 대응하기 위해 안티바이러스의 클라우드 기반 보호 기능을 활성화하여 알려지지 않은 변종 위협을 사전에 차단해야 한다. 또한 모든 엔드포인트 자산에 EDR을 배포하여 시스템 전반의 가시성을 확보하고 악의적인 활동에 대한 탐지와 대응 속도를 극대화해야 한다.

## 네트워크 통제 및 공격 표면 최소화

서버가 명시적으로 승인된 아웃바운드 트래픽만 허용하도록 기본 거부 형태의 Egress 필터링 모델을 도입하여, 악성 인프라와의 통신 및 데이터 외부 반출 시도를 원천 차단해야 한다. 이와 함께 시스템 내에 방치된 불필요한 소프트웨어와 관리 도구를 제거하여 공격자가 위장 수단으로 악용할 수 있는 공격 표면을 근본적으로 축소해야 한다.

## 모니터링 고도화 및 접근 제어 아키텍처 구현

웹 서버에 대한 상세한 로깅을 활성화하여 예상치 못한 파일 변경이나 의심스러운 웹 요청 등의 이상 징후를 상시 감시해야 한다. 동시에 엔터프라이즈 액세스 모델을 도입하여 무분별한 권한 상승을 억제하고 인프라 전반에 걸쳐 강력한 접근 통제를 강제해야 한다. 마지막으로 침해 사고 조사 과정에서 식별된 탐지 및 운영상의 사각지대를 즉각적으로 보완하여 SOC의 대응 역량을 한층 더 강화해야 한다.


# [Referenced]
* [Microsoft Incident Response](https://www.microsoft.com/en-us/security/blog/2026/05/12/undermining-the-trust-boundary-investigating-a-stealthy-intrusion-through-third-party-compromise/)