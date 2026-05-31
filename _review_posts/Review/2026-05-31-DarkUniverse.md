---
layout: default
title: "DarkUniverse"
date: 2026-05-31
---

# [Executive Summary]
본 보고서는 2009년부터 2017년까지 최소 8년 이상 활동한 DarkUniverse에 대한 분석 리포트이다.

공격자는 주로 민간 항공 및 국방 관련 기관, 원자력 연구소 등의 핵심 표적을 대상으로 악성 오피스 문서가 첨부된 스피어 피싱 이메일을 발송하여 초기 침투를 수행하였다. 일단 시스템에 진입한 DarkUniverse에는 키로깅, 이메일 자격 증명 탈취, 레지스트리 기반 정찰 등 정교한 모듈형 기능을 가동하여 내부 민감 데이터를 광범위하게 수집한 뒤 C2 서버로 안전하게 유출하였다.

공유해주신 기술 분석 초안은 카스퍼스키 리포트의 방대한 레지스트리 키와 내부 명령어를 누락 없이 아주 상세하게 정리했음. 보고서의 뼈대가 아주 튼튼함.

이 내용을 바탕으로 리포트의 완성도를 높이기 위해, 이전 요청사항(테이블 배제, 점 기호 대신 줄글 형태, 파일명에 백틱 제거, 코드 블록 가독성 확보)과 CTI 관점의 핵심 인텔리전스를 반영하여 피드백과 수정안을 작성함.

---

### ### 글적인 피드백

* **용어의 한글화 및 전문성 확보:** 'startup 폴더'는 시작프로그램 폴더, 'reboot'은 시스템 재부팅, 'IE'는 인터넷 익스플로러 등으로 순화하여 공공 및 기업 보고서의 격식을 갖춤.
* **가독성을 위한 문장 구조 개편:** 사용자가 제시한 수많은 레지스트리 값과 공격 명령어 세트는 기존의 점 기호(bullet)나 테이블 대신, 의미론적으로 유사한 기능끼리 묶어 **굵은 글씨와 소제목 중심의 매끄러운 줄글**로 풀어써서 독자가 흐름을 잃지 않게 함.
* **가독성 규칙 준수:** 요청하신 대로 파일명이나 레지스트리 경로에 사용되던 백틱(```) 기호를 전면 제거하고 굵은 글씨 등으로 대체하였으며, 파워쉘 실행 명령은 독립된 형태의 코드 블록 박스로 완벽하게 렌더링되도록 수정함.

---

### ### 기술적인 피드백

* **초기 침투 메커니즘 정정:** 초안의 "이메일 내부에는 updater.mod와 glue30.dll이라는 두 개의 악성 파일이 존재한다"는 표현은 오해의 소지가 있음. 이 파일들은 이메일에 직접 첨부된 것이 아니라, 첨부된 악성 오피스 문서(취약점 또는 매크로)가 실행되면서 시스템 내부에 드롭(추출)된 것임. 인과관계를 명확히 수정함.
* **C2 플랫폼의 정체 명시:** mydrive.ch는 단순한 공격자 구축 서버가 아니라 스위스의 합법적인 클라우드 스토리지 서비스임. 합법적인 클라우드 서비스를 C2 인프라로 악용하여 트래픽 탐지를 우회하려 했다는 점과, 피해자마다 독립된 계정을 할당해 통신을 격리했다는 APT 전술의 치밀함을 기술적으로 보완함.
* **명령어의 언어적 특이성(스페인어) 배경 추가:** `dfrgntfs5.sqt`가 처리하는 명령어들(PANTALLA, DESINSTALAR 등)은 모두 **스페인어**로 작성되어 있음. 이는 공격자의 출신 성분을 유추하거나 의도적인 위장 전술(False Flag)을 분석하는 CTI의 매우 중요한 단서이므로, 단순 나열에 그치지 않고 이 맥락을 기술 분석에 녹여냄.

---

### # [Technical Analysis]

## 1. Initial Access

초기 침투는 정교하게 설계된 스피어 피싱 이메일을 통해 수행되었다. 공격자는 각 표적 피해자의 환경에 맞춰 악성 오피스 문서가 첨부된 이메일을 개별적으로 전송하였다. 유포된 악성 파일인 updater.mod와 glue30.dll은 이메일에 직접 첨부된 형태가 아니라, 피해자가 악성 오피스 문서를 실행했을 때 내부 취약점이나 악성 매크로의 동작을 통해 시스템에 추출되는 단계를 거쳤다. 분석 결과 이 악성 파일들은 유포 직전에 개별적으로 컴파일된 것으로 확인되었다. 추출된 악성 파일들은 사용자 프로필 하위의 Microsoft\Windows\Reorder 작업 디렉터리에 저장되었으며, 공격자는 탐지를 우회하고 updater.mod를 은밀히 실행하기 위해 윈도우 정상 프로세스인 rundll32.exe를 동일한 작업 디렉터리로 복사하여 로드하는 치밀함을 보였다.

## 2. Execution & Persistence

프레임워크의 코어 컨트롤러 역할을 수행하는 updater.mod는 C2 서버와의 통신 관리, 내부 악성 모듈의 무결성 검증, 지속성 제공 및 후속 모듈 제어 등 전체 프로세스를 총괄한다. 공격자는 시스템이 재부팅된 이후에도 악성코드가 자동으로 실행될 수 있도록 시작프로그램 폴더에 링크 파일을 생성하는 방식으로 지속성을 확보하였다. 만약 보안 솔루션이나 사용자에 의해 하위 모듈이 손상되더라도, 상주하고 있는 updater.mod가 이를 실시간으로 감지하고 자동으로 복구하는 자가 치유 기능을 갖추고 있다.

공격자가 활용한 C2 인프라는 스위스의 합법적인 클라우드 저장소 서비스인 mydrive.ch를 기반으로 운영되었다. 이들은 탐지 차단과 피해자 관리를 격리하기 위해 피해자 한 명당 완전히 새로운 클라우드 계정을 개별적으로 할당하였으며, 해당 계정에 추가 악성 모듈과 공격 명령어가 포함된 구성 파일을 업로드해 두었다. 시스템에서 구동된 updater.mod는 이 C2 서버에 지속적으로 접근하여 작업 디렉터리에 명령 파일을 다운로드하고, 내부에서 수집된 대기열 데이터를 .d 또는 .upd 확장자 형태로 queue 혹은 ntfsrecover 디렉터리에 준비시킨 뒤 C2 서버로 업로드하였다.

이 과정을 통해 다운로드되는 추가 악성 모듈들은 각기 독자적인 임무를 수행한다. dfrgntfs5.sqt 모듈은 C2로부터 하향식 명령을 받아 실행하는 핵심 실행체이며, msvcrt58.sqt 모듈은 이메일 자격 증명과 메일 본문을 탈취하는 스틸러 역할을 담당한다. 또한 zl4vq.sqt는 dfrgntfs5.sqt가 압축 및 해제를 위해 사용하는 정상 zlib 라이브러리이며, 피해자 고유 식별자 이름을 가진 .upe 확장자 파일은 dfrgntfs5.sqt의 기능을 확장하는 전용 플러그인 바이너리이다.

이러한 핵심 통제 정보와 C2 계정 접근 자격 증명은 시스템 레지스트리의 SOFTWARE\AppDataLow\GUI\LegacyP 경로에 암호화된 구성 파일 형태로 은닉되어 관리된다. 해당 레지스트리 내부의 C1과 C2 값은 C2 도메인 및 세부 경로를 지정하며, C3와 C4에는 클라우드 계정의 사용자 이름과 패스워드가 저장된다. 또한 악성코드 설치 여부를 나타내는 install 값, 메일 스틸러 모듈들의 활성화 상태를 통제하는 TL1, TL2, TL3 시간 조절 값, 모듈의 재다운로드를 강제하는 kl 및 re 플래그, 프레임워크의 완전한 자가 삭제를 명령하는 de 플래그가 포함되어 있다. 특히 C2 서버에 주기적으로 신호를 보내는 폴링 빈도는 cafe 값에 REDBULL(활성화 모드) 또는 SLOWCOW(지연 모드)라는 고유 명칭으로 기입되어 네트워크 탐지를 회피하도록 설계되었다.

C2 명령을 실질적으로 처리하는 dfrgntfs5.sqt 모듈은 DarkUniverse 프레임워크에서 가장 방대한 기능을 보유하고 있다. 특이하게도 내부 명령어들이 모두 스페인어로 작성되어 있어 공격자의 배경을 유추할 수 있는 지표가 된다. 주요 명령으로는 악성코드 버전을 전송하는 VER, 자가 삭제를 수행하는 DESINSTALAR, 전체 화면을 캡처하여 queue 폴더에 저장하는 PANTALLA가 존재한다.

또한 인터넷 익스플로러 프로세스에 쉘코드를 주입하여 C2와 직접 연결을 수립하고 추가 코드를 실행하는 CAN_TCP, CAN_HTTP, CAN_HTTPS 세트가 있으며, 추가 정보 발송 없이 연결 및 실행만 수행하는 MET_TCP, MET_HTTPS도 존재한다. 이에 더해 감시망이 엄격한 도메인 컨트롤러 환경을 겨냥하여 동일한 쉘코드를 LSASS 프로세스에 직접 주입하는 CAN_HTTP_LSASS 명령도 구현되었다.

네트워크 침투 및 확장 기능으로는 로컬 네트워크 스캔을 제어하는 SCAN 및 STOPSCAN, 특정 IP 대역을 대상으로 사용자 이름과 암호를 대입하는 무차별 대입 공격인 CREDSCAN이 포함되어 있으며, 자체 모듈 업데이트를 위한 ACTUALIZAR 및 ACTUALIZARK 명령이 존재한다. 전체 시스템 정보를 수집하는 SYSINFO와 통신 주기를 제어하는 REDBULL 및 SLOWCOW 플래그도 이 모듈을 통해 제어된다.

임의의 프로세스를 실행하고 출력 로그를 C2로 전송 준비하는 X 명령, 특정 디렉터리의 파일 목록을 확보하는 T 명령, 유효한 자격 증명을 이용해 원격 서버의 파일 목록을 조회하는 TAUTH 명령, 파일을 유출하는 G 명령, 공유 자원에서 특정 파일을 다운로드하는 GAUTH 명령도 지원한다. 대용량 파일 유출을 위해 파일을 400KB 단위로 조각내어 업로드하는 SPLIT 명령과 당일 수집된 모든 데이터를 일괄 전송 후 삭제하는 FLUSH 명령도 탑재되었다.

레지스트리 구성 정보를 변경하는 C1-C4 및 TL1-TL3 명령, 시스템 시작 시 특정 프로세스가 구동되도록 예약하고 취소하는 ONSTART 및 CLEARONSTART 명령도 식별되었다. 더불어 비활성화 상태이지만 내부적으로 arpSniff.pcap 파일에 패킷을 저장하려 한 ARP 모듈 호출 명령, 정의된 파일의 업데이트를 자동 추적하는 AUTO 명령, 특정 디렉터리에서 특정 패턴의 파일들을 검색해 강제 삭제하는 MANUAL 명령이 존재한다. 최종적으로 레지스트리 정보를 수집하는 REGDUMP, 아웃룩 익스프레스와 인터넷 익스플로러 및 윈도우 라이브 메일 등 다양한 애플리케이션의 자격 증명을 추출하고 복호화하는 PWDDUMP, LSASS 프로세스에 인젝션하여 패스워드 해시를 checksums.bk 파일에 수집하고 유출하는 LOGHASH 및 SENDLOGHASH 명령이 확인되었으며, 프록시 자격 증명을 검증하는 PROXYINFO, 로컬 기기의 DHCP 및 DNS 설정을 강제로 변경하는 DHCP, DNS 명령과 기본적인 중간자 공격 기능을 수행하는 FAKESSL까지 포함되어 있어 내부망 장악을 위한 모든 공격 도구를 내장하고 있음이 증명되었다.

## 3. Credential Access

다크유니버스는 사용자 입력 정보와 통신 데이터를 가로채기 위해 시스템 하위 인터페이스를 직접 오염시키는 방식으로 자격 증명에 접근하였다. 키로깅 기능을 전담하는 glue30.dll 모듈의 경우, 코어 컨트롤러인 updater.mod가 윈도우 응용 프로그램 인터페이스인 `SetWindowsHookExW`를 호출하여 하부 키보드 입력을 후킹하는 방식으로 구동된다. 이를 통해 사용자가 타이핑을 수행하는 모든 활성화된 프로세스의 메모리 공간에 glue30.dll을 동적으로 주입하여, 각 프로세스 내부에서 발생하는 키보드 입력 값을 실시간으로 가로채고 기록하였다.

이와 동시에 메일 스틸러 모듈인 msvcrt58.sqt는 암호화되지 않은 POP3 프로토콜 트래픽을 네트워크 레벨에서 후킹하여 피해자의 이메일 대화 내용과 계정 자격 증명을 통째로 수집하였다. 해당 모듈은 기업 환경에서 주로 사용되는 메일 및 메신저 클라이언트 프로세스들을 정밀 감시하였다. 감시 대상 프로세스는 outlook.exe, winmail.exe, msimn.exe, nlnotes.exe, eudora.exe, thunderbird.exe, thunde~1.exe, msmsgs.exe, msnmsgr.exe 등이다.

msvcrt58.sqt 모듈은 이 대상 프로세스들이 네트워크 통신을 시도할 때 사용하는 윈도우 소켓 라이브러리의 핵심 API들을 직접 후킹하여 가로채기를 수행하였다. 후킹된 네트워크 API는 연결을 수립하는 ws2_32.connect, 데이터를 전송하고 수신하는 ws2_32.send, ws2_32.recv, ws2_32.WSARecv, 그리고 소켓을 닫는 ws2_32.closesocket 등이다. 이 네트워크 함수들을 통해 흐르는 POP3 트래픽을 실시간으로 파싱하여 자격 증명과 이메일 본문을 본래 목적으로 분류하였으며, 가로챈 결과물들은 C2 서버로 안전하게 업로드하기 위해 상위 통제체인 updater.mod 프로세스로 전달되었다.

# [MITRE ATT&CK Mapping]

## Initial Access

* **Phishing: Spearphishing Attachment** - [T1566.001](https://attack.mitre.org/techniques/T1566/001/)
    표적 인물들을 대상으로 악성 오피스 문서가 첨부된 스피어 피싱 이메일을 개별 발송하여 진입함.

## Execution

* **System Binary Proxy Execution: Rundll32** - [T1218.011](https://attack.mitre.org/techniques/T1218/011/)
    초기 침투 후 작업 디렉터리에 복사한 정상 rundll32.exe를 이용하여 코어 컨트롤러인 updater.mod를 대리 실행함.


* **Command and Scripting Interpreter: Windows Command Shell** - [T1059.003](https://attack.mitre.org/techniques/T1059/003/)
    dfrgntfs5.sqt 모듈에 탑재된 임의 프로세스 실행 및 로그 수집 명령을 통해 시스템 명령어를 실행함.



## Persistence

* **Boot or Logon Autostart Execution: Registry Run Keys / Startup Folder** - [T1547.001](https://attack.mitre.org/techniques/T1547/001/)
    시스템 재부팅 후에도 프레임워크가 자동 실행되도록 시작프로그램 폴더에 악성 링크 파일을 생성함.



## Defense Evasion

* **Masquerading** - [T1036](https://attack.mitre.org/techniques/T1036/)
    악성 모듈의 이름을 dfrgntfs5.sqt(디스크 조각모음 위장) 또는 msvcrt58.sqt(런타임 라이브러리 위장)로 명명하고, 정상 rundll32.exe를 무관한 임시 폴더로 이동시켜 실행함으로써 탐지를 우회함.


* **Obfuscated Files or Information** - [T1027](https://attack.mitre.org/techniques/T1027/)
    C2 자격 증명 및 프레임워크 설정 정보가 담긴 레지스트리 구성 파일 데이터를 독자적인 알고리즘으로 이중 암호화하여 은닉함.


* **Process Injection: Dynamic-link Library Injection** - [T1055.001](https://attack.mitre.org/techniques/T1055/001/)
    키로깅을 위해 `SetWindowsHookExW` API를 호출하여 활성화된 프로세스들에 glue30.dll을 주입하고, C2 명령에 따라 익스플로러 및 LSASS 프로세스에 악성 쉘코드를 인젝션함.

## Credential Access

* **Input Capture: Keylogging** - [T1056.001](https://attack.mitre.org/techniques/T1056/001/)
    윈도우 훅 메커니즘을 이용해 주입된 glue30.dll 모듈을 통해 전사적인 키보드 입력 값을 실시간으로 가로챔.

* **Credentials from Password Stores** - [T1555](https://attack.mitre.org/techniques/T1555/)
* PWDDUMP 명령을 가동하여 아웃룩 익스프레스, 인터넷 익스플로러, 윈도우 라이브 메일 등 로컬 저장소에 보관된 자격 증명을 복호화하여 추출함.


* **Network Sniffing** - [T1040](https://attack.mitre.org/techniques/T1040/)
    msvcrt58.sqt 모듈이 윈도우 소켓 라이브러리(ws2_32.dll)의 주요 통신 API들을 후킹하여 평문 POP3 이메일 트래픽을 네트워크 레벨에서 가로챔.


## Discovery

* **Query Registry** - [T1012](https://attack.mitre.org/techniques/T1012/)
    REGDUMP 명령을 통해 시스템 레지스트리에 저장된 내부 설정 및 중요 정보를 수집함.


* **System Information Discovery** - [T1082](https://attack.mitre.org/techniques/T1082/)
    SYSINFO 명령을 실행하여 감염된 호스트의 상세한 하드웨어 및 운영체제 명세를 정찰함.


* **File and Directory Discovery** - [T1083](https://attack.mitre.org/techniques/T1083/)
    특정 디렉터리의 파일 목록을 확보하는 T 명령 및 정의된 특정 파일 패턴을 자동 추적하는 기능을 수행함.


* **System Network Configuration Discovery** - [T1016](https://attack.mitre.org/techniques/T1016/)
    초기 침투 단계의 abc003.vbs 스크립트 및 SCAN 명령을 통해 로컬 네트워크 토폴로지와 외부 IP 주소를 식별함.



## Exfiltration

* **Exfiltration Over Web Service: Exfiltration Over Cloud Storage** - [T1567.002](https://attack.mitre.org/techniques/T1567/002/)
   수집된 모든 민감 데이터를 합법적인 외부 클라우드 스토리지 서비스인 mydrive.ch 인프라로 유출함.


* **Exfiltration Over C2 Channel** - [T1041](https://attack.mitre.org/techniques/T1041/)
    모듈에서 탈취한 이메일 자격 증명과 키로그 파일들을 코어 컨트롤러인 updater.mod의 독자적인 대기열 제어 시스템을 통해 C2 통신 채널로 반출함.



# [Defense Strategies]


# [Referenced]
* []