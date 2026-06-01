---
layout: default
title: "Inversify"
date: 2026-06-01
---

# Inversify (InversifyJS)

Inversify는 타입스크립트를 위한 강력하고 가벼운 **의존성 주입(Dependency Injection, DI) 컨테이너**다. 소프트웨어 아키텍처에서 클래스 간의 결합도를 낮추고 유지보수성과 확장성을 극대화하기 위해 사용되는 핵심 도구다.

## 1. 의존성 주입(DI)이란 무엇인가?

일반적으로 객체를 생성할 때 필요한 다른 객체(의존성)를 내부에서 직접 생성(예: `new Service()`)하는 방식을 사용한다. 하지만 시스템 규모가 커지면 서비스 간의 결합도가 매우 높아져, 하나의 클래스를 수정할 때 이를 참조하는 모든 클래스를 찾아 변경해야 하는 문제가 발생한다.

의존성 주입은 객체가 필요한 의존성을 스스로 생성하는 대신, **외부(컨테이너)로부터 주입받는 패턴**이다. 이를 통해 클래스는 구현체가 아닌 인터페이스에만 의존하게 되어 코드 수정이 훨씬 유연해진다.

## 2. Inversify가 제공하는 핵심 기능

* **IoC (Inversion of Control) 컨테이너**: 애플리케이션의 모든 의존성을 관리하는 중앙 저장소 역할을 한다. 개발자는 컨테이너에 클래스와 인터페이스를 등록하기만 하면, 컨테이너가 알아서 인스턴스를 생성하고 필요한 곳에 주입해 준다.
* **데코레이터 기반 구성**: 타입스크립트의 데코레이터(`@injectable()`, `@inject()`)를 사용하여 간결하고 직관적으로 의존성을 선언할 수 있다.
* **타입 안전성(Type Safety)**: 타입스크립트의 인터페이스를 사용하여 의존성을 정의하므로, 런타임뿐만 아니라 컴파일 타임에도 강력한 타입 검증이 가능하다.

## 3. 실제 적용 구조 예시

Inversify를 사용하여 클래스 간의 결합도를 낮추는 과정은 다음과 같다.

#### [1. 인터페이스 및 클래스 정의]
인터페이스를 통해 추상화 계층을 만들고, `@injectable()` 데코레이터를 사용하여 컨테이너가 해당 클래스를 관리할 수 있게 지정한다.

```typescript
import { injectable } from 'inversify';

interface IParser {
    parse(code: string): any;
}

@injectable()
class JavaScriptParser implements IParser {
    public parse(code: string) {
        return "Parsed AST";
    }
}

```

#### [2. 의존성 주입]
의존 객체가 필요한 곳에서 직접 생성하지 않고, `@inject()` 데코레이터를 통해 컨테이너로부터 주입받는다.

```typescript
@injectable()
class MainProcessor {
    constructor(
        @inject('IParser') private parser: IParser
    ) {}

    public process(code: string) {
        return this.parser.parse(code);
    }
}

```

#### [3. 컨테이너 설정 및 바인딩]
어떤 인터페이스에 어떤 구현 클래스를 매핑할지 결정하는 바인딩 작업을 수행한다.

```typescript
import { Container } from 'inversify';

const container = new Container();
container.bind<IParser>('IParser').to(JavaScriptParser);
container.bind<MainProcessor>(MainProcessor).toSelf();

const processor = container.get(MainProcessor);

```
