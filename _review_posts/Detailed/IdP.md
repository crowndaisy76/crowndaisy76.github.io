---
layout: default
title: IdP
date: 2026-05-24
---

# [IdP (Identity Provider)]

IdP는 디지털 환경에서 사용자의 신원을 확인하고 인증 서비스를 제공하는 중앙 집중식 시스템이다. 클라우드와 SaaS 도입이 보편화된 현대 엔터프라이즈 환경에서, 사용자가 수많은 서비스마다 계정을 따로 만들지 않고 하나의 자격 증명으로 안전하게 접근할 수 있도록 인프라의 핵심 관문 역할을 수행한다.


## **1. 기술적 메커니즘 및 동작 원리**

IdP는 크게 두 가지 핵심 표준 프로토콜인 SAML 2.0(Security Assertion Markup Language)과 OIDC(OpenID Connect)를 기반으로 서비스 공급자(SP, Service Provider: 예: SharePoint, Salesforce 등)와 통신한다.

* **인증 릴레이 구조:** 사용자가 특정 SaaS(SP)에 로그인을 시도하면, 해당 서비스는 자체적으로 패스워드를 검증하지 않고 사용자를 신뢰 관계가 맺어진 IdP 로그인 페이지로 리디렉션한다.
* **토큰(Token) 발급:** IdP가 사용자의 신원(패스워드, 생체 인증, MFA 등)을 성공적으로 검증하면, 사용자 신원 정보가 디지털 서명된 보안 토큰(SAML Assertion 또는 JWT 토큰)을 발행하여 브라우저를 통해 SP로 전달한다.
* **접근 허가:** SP는 토큰의 위변조 여부와 서명을 검증한 후 사용자의 로그인 세션을 생성한다. 이 일련의 과정을 통해 사용자는 한 번의 로그인으로 여러 시스템에 접근하는 SSO(Single Sign-On)를 누릴 수 있다.


## **2. 대표적인 공급 솔루션 분류**

* **클라우드 고유 IDaaS (Identity as a Service):** Okta, Ping Identity, OneLogin 등 클라우드 환경에서 전사적 계정 통합 관리를 전문으로 제공하는 독립형 솔루션이다.
* **빅테크 생태계 통합형:** Microsoft Entra ID(구 Azure AD), Google Workspace IDP 등 자체 오피스/인프라 생태계와 긴밀하게 결합되어 기업 계정 관리를 수행하는 플랫폼이다.
