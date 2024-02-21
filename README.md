# 스프링MVC 2편 - 예외처리와 오류페이지, API 오류처리
> 김영한 강사님의 스프링MVC 2편 내용 정리 (목차보단 자세하고 PDF 보단 간략한 정리)

***
# 1. 예외처리와 오류페이지
***
# 1.1. 서블릿 예외처리 - 시작
##### 서블릿은 2가지 방식으로 예외처리를 지원
* `Excepiton`(예외)
* `response.sendError(HTTP 상태코드, 오류 메시지)`

### 1.1.2. Exception(예외)
```java
//예
throw new RuntimeException("예외발생!")
```
##### 자바 직접 실행
* 자바의 메인 메서드를 직접 실행하면 `main`이라는 이름의 쓰레드가 실행된다.
* 실행중 예외를 잡지 못하면, `main()` 메서드를 넘어서 예외 정보를 남기고 해당 쓰레드는 종료된다.

##### 웹 애플리케이션
* 웹 애플리케이션은 사용자 요청 별로 별도의 쓰레드가 할당되고, 서블릿 컨테이너 안에서 실행된다.
* 애플리케이션에서 try~catch로 예외를 잡아서 처리하면 문제가 없다.
* 애플리케이션에서 예외를 잡지 못하고 서블릿 밖으로 까지 예외가 전달되면 아래와 같이 동작한다.
  + `WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)`

### 1.1.3. response.sendError(HTTP 상태코드, 오류메시지)
```java
response.sendError(HTTP 상태 코드)
response.sendError(HTTP 상태 코드, 오류 메시지)
```
##### sendError 흐름
* WAS(sendError 호출 기록 확인) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러
* `response.sendError()`를 호출하면 `response` 내부에는 오류가 발생했다는 상태를 저장해둔다.
* 서블릿 컨테이너는 고객에게 응답하기 전에 `response`에 `sendError()`가 호출되어 있는지 확인한다.
* 호출되었다면 오류코드에 맞춰 오류페이지를 보여준다.

***
# 1.2. 서블릿 예외처리 - 오류 화면 제공
* 서블릿은 Exception(예외)가 발생해서 서블릿 밖으로 전달되거나 또는 `response.sendError()`가 호출되었을 때 각각의 상황에 맞춘 오류 처리 기능을 제공한다.

### 1.2.1. 서블릿 오류 페이지 등록
* `WebServerFactoryCustomizer<ConfigurableWebServerFactory>`를 implement해서 factory에 URL 경로, 오류페이지를 더한다.
* Controller를 통해서 URL을 매핑해주고 뷰를 연결해준다.

***
# 1.3. 서블릿 예외 처리 - 오류페이지 작동 원리
### 1.3.1. 작동 원리
##### 예외 발생 흐름
```
WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)
```
##### sendError 흐름
```
WAS(sendError 호출 기록 확인) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러
(response.sendError())
```
##### WAS는 해당 예외를 처리하는 오류 페이지 정보를 확인한다.
```
new ErrorPage(RuntimeException.class, "/error-page/500")
```
> 예를 들어서 `RuntimeException` 예외가 WAS까지 전달되면, WAS는 오류 페이지 정보를 확인한다. 확인해보니`RuntimeException` 의 오류 페이지로 `/error-page/500` 이 지정되어 있다. WAS는 오류 페이지를 출력하기 위해 `/error-page/500` 를 다시 요청한다.

##### 오류 페이지 요청 흐름
```
WAS `/error-page/500` 다시 요청 -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러(/error-page/500) -> View
```

##### 예외 발생과 오류페이지 요청 흐름
```
1. WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)
2. WAS `/error-page/500` 다시 요청 -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러(/error-page/500) -> View
```
> 여기서 웹브라우저(클라이언트)는 서버 내부에서 이런 로직을 전혀 모르고 서버 내부에서만 오류페이지를 찾기 위해 추가적인 호출을 한다.


### 1.3.2. 오류 정보 추가
* WAS는 오류페이지를 요청하는 것 뿐만아니라, 오류 정보를 `request`의 `attribute`에 추가해서 넘길 수 있다.
* 필요하면 오류정보를 오류페이지에서 사용할 수 있다.

##### request.attribute에 서버가 담아준 정보
* `javax.servlet.error.exception` : 예외
* `javax.servlet.error.exception_type` : 예외 타입
* `javax.servlet.error.message` : 오류 메시지
* `javax.servlet.error.request_uri` : 클라이언트 요청 URI
* `javax.servlet.error.servlet_name` : 오류가 발생한 서블릿 이름
* `javax.servlet.error.status_code` : HTTP 상태 코드
> SpringBoot 3.0 이상 부턴 javax가 아닌 jakarta로 변경되었다.


***
# 1.4. 서블릿 예외 처리 -필터
### 1.4.1. 예외처리에 따른 필터 DispatchType 이해

##### 예외 발생과 오류페이지 요청흐름
```
1. WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)
2. WAS `/error-page/500` 다시 요청 -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러(/error-page/500) -> View
```

* 한번 필터, 인터셉터에서 체크를 끝냈기 때문에 한번 더 호출되는 것은 비효율적이다.
* 결국 클라이언트로 부터 발생한 정상 요청인지, 오류 페이지를 출력하기 위한 내부 요청인지 구분하는 정보를 `DispatcherType`을 통해 제공 받는다.

##### DispatcherType (enum타입)
* `REQUEST` : 클라이언트 요청
* `ERROR` : 오류 요청
* `FORWARD` : MVC에서 배웠던 서블릿에서 다른 서블릿이나 JSP를 호출할 때 `RequestDispatcher.forward(request, response);`
* `INCLUDE` : 서블릿에서 다른 서블릿이나 JSP의 결과를 포함할 `RequestDispatcher.include(request, response);`
* `ASYNC` : 서블릿 비동기 호출

##### 필터와 DispatcherType
* `filterRegistrationBean.setDispatcherTypes(DispatcherType.REQUEST,DispatcherType.ERROR);`
* 위 코드로 필터에서 클라이언트 요청, 오류페이지 요청일 때 필터가 호출할지 결정할 수 있다.
* 기본 값이 DispatcherType.REQUEST이다.
* 특별히 오류 페이지 경로도 필터를 적용할 것이 아니면 기본 값을 그대로 사용하면 된다.

### 1.4.2. 인터셉터 중복 호출 제거
* 앞서 필터의 경우는 필터를 등록할 때 어떤 `DispatcherType`인 경우에 필터를 적용할지 선택할 수 있다.
* 인터셉터는 서블릿이 제공하는 기능이 아니라 스프링이 제공하는 기능이기 때문에 `DispatcherType`과 무관하게 항상 호출된다.
* 인터셉터의 경우는 요청 경로에 추가하거나 제외하기 쉽게 되어 있어서 `excludePathPatterns`를 사용해 뺴주면된다.

***
# 1.5. 스프링 부트 - 오류페이지1
### 1.5.1. 지금까지의 예외처리 페이지
* 예외 처리 페이지를 만들기 위해 아래와 같은 복잡한 과정을 거쳤다
  + `WebServerCustomizer`를 만들고
  + 예외 종류에 따라서 `ErrorPage`를 추가하고 예외 처리용 컨트롤러 `ErrorPageController`를 만들었다.
 
### 1.5.2. 스프링 부트 예외처리 페이지
##### 스프링 부트는 이런 과정을 모두 기본으로 제공한다.
* `ErrorPage`를 자동으로 등록한다. 이때 `/error`라는 경로로 기본 오류 페이지를 설정한다.
  + `new ErrorPage("/error")` , 상태코드와 예외를 설정하지 않으면 기본 오류 페이지로 사용된다.
  + 서블릿 밖으로 예외가 발생하거나, `response.sendError(...)` 가 호출되면 모든 오류는 `/error` 를 호출하게 된다.
 
* `BasicErrorController`라는 스프링 컨트롤러를 자동으로 등록한다.
  + `ErrorPage`에서 등록한 `/error`를 매핑해서 처리하는 컨트롤러이다.
 
##### 주의점
* 수동 등록한 것이 언제나 우선이기 때문에 `WebServerCustomizer`의 `@Component`를 주석 처리해야지 스프링 부트가 제공하는 기본 오류 메커니즘을 사용할 수 있다.
##### 개발자는 오류페이지만 등록
* `BasicErrorController` 는 기본적인 로직이 모두 개발되어 있다.
* 개발자는 오류 페이지 화면만 `BasicErrorController` 가 제공하는 룰과 우선순위에 따라서 등록하면 된다.
* 정적 HTML이면 정적 리소스, 뷰 템플릿을 사용해서 동적으로 오류 화면을 만들고 싶으면 뷰 템플릿 경로에 오류 페이지 파일을 만들어서 넣어두기만 하면 된다.

##### 뷰 선택 우선순위
* BasicErrorController` 의 처리 순서
1. 뷰 템플릿
  + `resources/templates/error/500.html`
  + `resources/templates/error/5xx.html`
2. 정적 리소스( `static` , `public` )
  + `resources/static/error/400.html`
  + `resources/static/error/404.html`
  + `resources/static/error/4xx.html`
3. 적용 대상이 없을 때 뷰 이름( `error` )
  + `resources/templates/error.html`

***
# 1.6. 스프링 부트 - 오류페이지2
### 1.6.1. BasicErrorController가 제공하는 기본 정보
```
* timestamp: Fri Feb 05 00:00:00 KST 2021
* status: 400
* error: Bad Request
* exception: org.springframework.validation.BindException
* trace: 예외 trace
* message: Validation failed for object='data'. Error count: 1
* errors: Errors(BindingResult)
* path: 클라이언트 요청 경로 (`/hello`)
```
* 오류 관련 내부 정보들은 고객에게 노출하는 것은 좋지 않다. 이러한 오류 정보를 `model`에 포함할지 여부를 선택할 수 있다.
```
//application.properties
server.error.include-exception=false //exception 포함 여부( `true` , `false` )
server.error.include-message=never //message 포함 여부
server.error.include-stacktrace=never //trace 포함 여부
server.error.include-binding-errors=never //errors 포함 여부
```
### 1.6.2. 확장 포인트
에러 공통 처리 컨트롤러의 기능을 변경하고 싶다면 `ErrorController` 인터페이스를 상속 받아서 구현하거나 `BasicErrorController` 상속 받아서 기능을 추가하면 된다.


***
# 2. API 예외 처리
***
### 1.1. API 예외 처리 - 시작
* HTML 페이지의 경우는 오류페이지만 있으면 대부분의 문제는 해결할 수 있다.
* 그에 반해 API의 경우에는 오류 상황에 맞는 오류 응답 스팩을 정하고, JSON으로 데이터를 내려주어야한다.
* `@RestController`를 붙이면 결과는 JSON으로 보내지지만, 오류는 오류페이지로 자동 포워딩 된다.
* 





