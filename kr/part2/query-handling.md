질의 요청 처리
==============

질의(query) 처리를 위한 컴포넌트는 애플리케이션으로 들어오는 조회 요청 메시지를 처리합니다. 이벤트 리스터가 생성한 일반적으로 뷰(view)모델로부터 데이터를 읽어 옵니다. 질의 처리 컴포넌트는 일반적으로 이벤트나 명령을 발생시키지 않습니다.

질의 요청 처리자 정의하기
=======================

Axon내에서, 질의 처리를 위한 다수의 메서드를 선언할 수도 있습니다. 해당 메서드에는 `@QueryHandler` 에노테이션을 붙여 사용합니다. 메서드가 처리할 메시지는 매개변수를 통해 선언합니다.

기본적으로, `@QueryHandler` 에노테이션을 붙인 메서드들은 다음과 같음 매개변수들을 선언할 수 있습니다.

- 첫 번째 매개변수로 질의 메시지의 페이로드를 선언할 수 있습니다. 만약 `@QueryHandler` 에노테이션에 핸들러가 처리할 질의명을 명시적으로 선언한다면, 첫 번째 매개변수를 `Message` 혹은 `QueryMessage` 타입으로 선언할 수 도 있습니다. 기본적으로 질의 메시지의 페이로드 클래스 명을 쿼리명으로 사용합니다.

- `@MetaDataValue` 에노테이션이 붙은 매개변수는 에노테이션에 명시된 키에 해당하는 메타 데이터값으로 해석되어 메서드로 전달됩니다. `@MetaDataValue` 에노테이션에 `required` 속성을 기본값인 `false`로 주게 되면, 없는 메타 데이터를 `null`로 처리하여 넘겨줍니다. 반면 `required`가 `true`이고 해당 메타 데이터가 존재하지 않으면, 해당 메서드는 호출되지 않습니다.

- `MetaData` 타입의 매개변수는 주입된 `CommandMessage`의 전체 `MetaData`를 포함하게 됩니다.

- `UnitOfWork` 타입의 매개변수를 통해 현재의 작업 단위 객체를 주입 받을 수 있습니다. 이를 통해, 쿼리 처리자는 작업 단위의 특정 단계에서 실행해야 할 작업을 등록할 수 있습니다. 혹은 현재의 작업 단위 객체에 등록된 자원을 사용할 수 있습니다.

- `Message` 혹은 `QueryMessage` 타입의 매개변수는 페이로드와 메타 데이터를 포함한 전체 메시지를 주입 받을 수 있습니다. 여러 메타 데이터가 필요한 경우 혹은 메시지의 다른 속성이 필요할 때 유용하게 사용할 수 있습니다.

`ParameterResolverFactory` 인터페이스를 직접 구현하고 `/META-INF/service/org.axonframework.common.annotation.ParameterResolverFactory` 파일을 생성한 후 구현체 클래스를 설정하여 `ParameterResolver`들을 추가로 설정할 수 있습니다. 더 상세한 내용은 [고급 사용자 지정](../part4/advanced-customizations.md)을 참고하세요. // TODO: 고급 사용자 지정 링크 수정.

모든 상황에서, 질의 처리자 인스턴스마다 하나의 이벤트 처리 메서드가 호출이 됩니다. Axon은 다음과 같은 규칙을 적용하여 가장 적합한 호출 대상 메서드를 찾아냅니다.

1. 실제 인스턴스의 클래스(this.getClass()로 반환되는 클래스) 계층 구조상에서, `@QueryHandler ` 에노테이션이 붙은 메서드들을 평가합니다.

2. 모든 매개변수들을 값으로 치환할 수 있는, 하나  이상의 메서드가 발견되었을 경우, 가장 구체적인 타입을 선언한 메서드를 선택합니다.

3. 현재 클래스 구조상에서 메서드를 발견할 수 없는 경우, 상위 타입의 클래스 구조를 같은 방법으로 탐색합니다.

4. 클래스 구조상의 최상위에서도 메서드를 발견할 수 없는 경우, 해당 이벤트는 무시됩니다. 명령 처리와 유사하지만, 이벤트 처리와는 달리 질의 처리는 질의 메시지의 클래스 계층을 고려하지 않는 점에 유의하세요.

명령 처리와 유사하지만, 이벤트 처리와는 달리 질의 처리는 질의 메시지의 클래스 계층을 고려하지 않는 점에 유의하세요.

```java
// QueryB는 QueryA를 상속한다고 가정합니다.
// QueryC는 extends를 상속한다고 가정합니다.
// SubHandler의 단일 인스턴스가 등록되어 있다고 가정합니다.

public class TopHandler {

    @QueryHandler
    public MyResult handle(QueryA query) {
    }

    @QueryHandler
    public MyResult handle(QueryB query) {
    }

    @QueryHandler
    public MyResult handle(QueryC query) {
    }
}

public class SubHandler extends TopHandler {

    @QueryHandler
    public MyResult handleEx(QueryB query) {
    }
}
```

위의 예제에서, `QueryB` 타입의 질의 처리 시, SubHandler의 쿼리 처리자 메서드가 호출되고 `MyResult`를 반환합니다. `QueryB` 타입의 질의를 처리할 때는 `TopHandler`의 메서드가 호출되고 `MyResult`를 반환합니다.

질의 처리자 등록하기
==========================

같은 질의 명과 응답 타입(유형)에 대한 다수의 쿼리 처리자를 등록할 수 있습니다. 질의들을 처리할 때, 클라이언트는 하나 질의 처리자로부터 결과를 받을지 혹은 사용 가능한 모든 질의 처리자로부터 결과를 받을지를 지정할 수 있습니다.

스프링 프레임워크와 연동하여 사용하기
------------
스프링의 자동설정(AutoConfiguration)을 사용하면, `@QueryHandler` 에노테이션이 붙은 메서드들을 찾기 위해 모든 싱글 톤(singleton) 빈(bean)들을 검색합니다. 발견된 각각의 메서드들은 새로운 질의 처리자로 질의 버스(query bus)에 등록됩니다.

설정 API 사용하기
-----------------------
설정 API를 사용하여 질의 처리자를 등록할 수 있습니다. 아래의 예제 코드처럼 `Configurer` 클래스의 `registerQueryHandler` 메서드를 사용하면 됩니다.

```java
// 샘플 쿼리 처리자
public class MyQueryHandler {
    @QueryHandler
    public String echo(String echo) {
        return echo;
    }
}

...

// 쿼리 처리자 등록하기
Configurer axonConfigurer = DefaultConfigurer.defaultConfiguration()
    .registerQueryHandler(conf -> new MyQueryHandler);
```
