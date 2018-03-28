고급 사용자 정의
=======================

매개변수 리졸버
-------------------

`ParameterResolverFactory`를 상속하고 구현 클래스의 패키지 명을 포함한 전체 이름을 포함하는 `/META-INF/service/org.axonframework.common.annotation.ParameterResolverFactory`파일을 생성하여 추가적인 `ParameterResolver`를 설정할 수 있습니다.

> **주의**
>
> 현재 OSGi 지원은 필수 헤더들이 매니페스트 파일에 기술되어야 한다는 것으로 제한됩니다. `ParameterResolverFactory`의 인스턴스들은 OSGi상에서 자동으로 감지되지만, 클래스 로더의 제한들 때문에, `/META-INF/service/org.axonframework.common.annotation.ParameterResolverFactory` 파일의 내용을 매개변수들을 처리하는 클래스들을 포함하는 OSGi 번들에 복사해야 할 수도 있습니다. (예, 이벤트 처리자)

메타 에노테이션
----------------

작성 예정

메시지 처리자 기능 재정의
------------------------------------

작성 예정
