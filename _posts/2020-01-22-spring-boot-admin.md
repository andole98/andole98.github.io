---
title: "스프링 부트 어드민 사용해보기"
category: Spring
---

Spring Boot Admin, SBA라고도 한다.  
어드민 구성을 위해서 둘 이상의 스프링 부트 프로젝트가 필요하다.  
하나는 어드민이고, 나머지는 어드민이 **모니터링하는 대상**이다.  
모니터링 환경을 구축하려면 서로 다른 인스턴스를 활용하는게 맞겠지만, 인프라 ~~money~~ 가 없으니 **로컬에서 어드민과 타겟, 두 프로젝트**를 띄워 보자.  
Gradle 멀티 모듈 프로젝트로 어드민과 모니터링 타겟을 구성한다.

## 개요
스프링 부트 어드민은 `actuator` 기반으로 데이터를 분석한다. 따라서 모니터링 타겟은 actuator가 활성화 되어 있어야 하고, 적절한 엔드포인트가 구성되어 있어야 한다.
스프링 부트 **어드민** 은 스스로 타겟을 탐지하지 않는다. **모니터링 타겟** 이 어드민에게 스스로를 등록해야 한다. 여기서 많이 헤멨는데, 내 생각에는 어드민이 타겟을 탐지해서 모니터링을 수행할 줄 알았다. 반대로 타겟이 어드민을 알아내서 스스로를 등록하는 방식이다.  
어드민 쪽에서는 `admin-server`, `admin-server-ui` 패키지가 필요하다. 타겟 쪽에서는 `admin-clinet`, `actuator` 패키지가 필요하다.  

## 멀티모듈 구성
멀티 모듈 프로젝트 구성은 검색하면 잘 나온다. 여기서 루트 프로젝트는 `monitoring-practice`, 어드민 모듈은 `admin`, 모니터링 어플리케이션은 `monitoring-target`으로 이름지었다.

{% gist andole98/315930e58a276e760d50386b71f0a97d %}

dependency에 필요한 패키지들을 추가해서 구성하면 된다.

주의할 점은, 스프링 부트 버전이 2.2.x 일 때 스프링 부트 어드민은 2.2.0으로 잡아야 한다. 다른 버전으로 구성하면 `adminHandlerMapping` 빈을 생성할 때 스택오버플로 예외가 발생한다. [이슈](https://github.com/codecentric/spring-boot-admin/issues/1279)

## 설정
### Admin Side
어드민은 어드민 환경 구성에 대한 설정이 필요하다. 스프링 부트 프로젝트는 기본 포트로 8080을 사용한다. 어드민과 모니터링 타겟이 같은 포트를 사용할 수 없으므로 둘 중 하나의 포트를 바꿔줘야 한다. 여기서는 **어드민 포트를 8000**, **타겟 포트를 8080**으로 구성한다. 

{% gist andole98/60bb22c354429a34d352f2c2710abb8b %}

모듈 실행 후 http://localhost:8000 으로 접속해보자. 다음과 같은 화면이 나온다.

![admin](https://user-images.githubusercontent.com/40727649/72961958-41002880-3df6-11ea-8cab-da8ce1be7ea6.png)

애플리케이션이 0으로 나온다. 모니터링 타겟을 구성해서 어드민이 추적할 수 있게 해야 한다. 

### Target Side
모니터링 대상에서는 몇 가지 설정이 필요하다. 

{% gist andole98/f7938db45325d5068d5e9aca68d47f7d %}

어드민은 actuator 를 사용하므로 엔드포인트를 열어줘야 한다. 모든 엔드포인트를 노출했고 actuator 정보도 디테일로 세팅했다.  
모니터링 타겟을 어드민에 등록하려면, 어드민 url을 지정해줘야 한다. `spring.boot.admin.client.url` 에 어드민 url을 설정한다. 위에서 8000포트로 잡았으므로 http://localhost:8000이다. 

## 모니터링
필수 설정은 끝났다. `target` 모듈을 실행하면 (어드민과 타겟 모두 실행된 상태) 어드민에서 어플리케이션을 볼 수 있다. 

![app](https://user-images.githubusercontent.com/40727649/72962523-ee277080-3df7-11ea-9536-cd0740b9f6b1.png)

![dashboard](https://user-images.githubusercontent.com/40727649/72962524-eec00700-3df7-11ea-814c-5f1db7771961.png)

어드민에서는 유용한 정보를 제공한다. jvm 관련 항목도 있고 다양한 메트릭을 구성할 수도 있다. 로깅도 관리할 수 있다. 

![metric](https://user-images.githubusercontent.com/40727649/72962757-a05f3800-3df8-11ea-88d8-33d34fc42b51.png)

모니터링 타겟에 아무 기능이 없으므로 별 게 없다. 모니터링 하고 싶은 앱에서 어드민을 설정하고 모니터링 하면 여러 설정값이나 로깅을 실시간으로 볼 수 있다.

## Gradle parallel

어드민은 필수적으로 둘 이상의 서버가 필요하다. 프로덕션 환경에선 위처럼 설정하지 않겠지만, 로컬 개발 환경에서는 충분히 해볼 만하다고 생각한다. 다만 둘 이상의 모듈을 실행해야 하는게 귀찮다. 모듈 하나씩 찾아가서 실행하는게 번거롭다.  

Gradle은 여러 태스크를 동시에 수행할 수 있다. 간단한 gradle 스크립트를 구성했다.

{% gist andole98/99900fa5d5b9d65bd5ae9da7c5c019fc %}

어드민과 타겟을 동시에 시작하려면 
```bash
./gradlew start
```

중지하려면 

```bash
./gradlew -stop
```

어드민을 구성했으니 지금까지 만들어 둔 프로젝트들을 모니터링 해보고 싶어진다. 그러나 실제로 배포된 서비스가 아니라면, 프로젝트는 놀고 있을 것이다. 놀고 있는 서버를 모니터링 하는 것도 웃기니까 부하 테스트와 연결하고 모니터링을 수행해 보자.

다음 포스트에 네이버 `nGrinder` 를 사용해서 부하테스트를 수행하고, 어드민에서 여러 지표들을 모니터링 해본다.
