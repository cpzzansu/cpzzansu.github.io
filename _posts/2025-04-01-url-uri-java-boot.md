---
title: Java에서 URL 생성
description: >-
  URL(String) 생성자의 Deprecation과 URI의 활용
author: author_js
date: 2025-04-01 12:49:00 +0900
categories: [JAVA, Spring Boot]
tags: [java, spring, boot, 자바, 스프링부트, 스프링, url, uri]
pin: true
---

토스 페이먼트를 프로젝트에 연동을 하면서, 갑자기 자바의 실행 창에서 아래와 같은 메시지를 나타내기 시작했다.

```java
BUILD SUCCESSFUL in 811ms
2 actionable tasks: 1 executed, 1 up-to-date
Note: /yourFile.java uses or overrides a deprecated API.
Note: Recompile with -Xlint:deprecation for details.
오후 4:39:23: 실행이 완료되었습니다 ':classes'.
```

여기서 봤을때는 그냥 코드지만, 인텔리제이의 화면에서는 Note 부분이 빨간색으로 보여서 뭔가 심각한 문제처럼 보인다. `-Xlint:deprecation` 을 사용해서
더 자세하게 어떤 부분을 바꿔야하는지 알 수 있다.

`build.gradle` 파일에 아래의 코드를 추가해주고 다시 실행해주면 더 자세하게 어떤 것을 바꿔야 하는 지 알 수 있다.

```java
tasks.withType(JavaCompile) {
    options.compilerArgs << "-Xlint:deprecation"
}
```
```java
BUILD SUCCESSFUL in 1s
2 actionable tasks: 1 executed, 1 up-to-date
/yourFile.java:79: warning: [deprecation] URL(String) in URL has been deprecated
    URL url = new URL("https://api.tosspayments.com/v1/payments/confirm");
              ^
1 warning
오후 4:51:03: 실행이 완료되었습니다 ':classes'.
```

토스 페이먼트로 확인을 보내기 위해 URL을 추가해주는 부분에서 바꿔야하는 부분이 나왔다.

최근 JAVA 개발 환경에서 URL 객체를 생성할 때 URL(String) 생성자를 사용하면 이런 경고 메시지를 볼 수 있다고 한다. 이 문구는
해당 생성자가 deprecated(더 이상 사용 권장되지 않음) 되었기 때문이다. 이번 글에서는 그 이유와 대체 방법, 그리고 코드 작성 시
고려해야 할 안정성과 명확성에 대해서 알아보도록 하자.

## URL(String) 생성자가 Deprecated된 이유

### 1.1 안정성 문제

URL(String) 생성자는 단순히 문자열을 입력받아 URL 객체를 생성한다. 그런데 이 방식은 URL 문자열의 구문을 충분히 검증하지 않는다.

- 문자열 검증 부족 : 잘못된 형식의 문자열을 전달할 경우, 런타임에 예기치 않은 동작이나 예외가 발생할 가능성이 있다.
- 보안 문제 : 제대로 검증되지 않은 입력이 URL 객체로 변환되면, 이후 네트워크 통신에서 의도치 않은 결과를 초래할 수 있다.

### 1.2 명확성 부족

URL은 여러 구성 요소(프로토골, 호스트, 포트, 경로 등)로 이루어져 있다.
- 구성 요소 구분 미흡 : 단순 문자열로 URL을 처리하면, 각 요소를 구분하기 어렵다.
- 가독성 저하 : 코드 상에서 URL의 각 부분을 개별적으로 다루거나, 후에 수정할 때 어떤 값이 어떤 의미인지 파악하기 어렵다.

## URI를 활용한 대체 방법

대신, JAVA에서는 URI 클래스를 사용하여, URL을 구성하는 각 요소를 명확하게 구분하고 검증할 수 있다. URI 클래스는 다음과 같은 장점을 제공한다.
- 엄격한 구문 검증 : URI 문자열을 파싱할 때, 각 구성요소가 올바른지 검증하여 잘못된 입력을 조기에 발견할 수 있다.
- 구성 요소의 분리 : 스킴, 호스트, 포트, 경로 등 URL의 각 부분을 명확하게 관리할 수 있어 코드의 가독성이 향상된다.

아래와 같이 바꾸면 된다.

```java
URI uri = URI.create("https://api.tosspayments.com/v1/payments/confirm");
URL url = uri.toURL();
```

## 결론

Java에서 네트워크 통신이나 외부 API 호출을 위해 URL 객체를 생성할 때, 단순히 URL(String) 생성자를 사용하는 대신 URI 클래스를 활용하는 것이 더욱 안전하고 명확한 코드를 만드는 방법이 된다.
•	안전성: 잘못된 형식의 URL로 인한 문제를 미연에 방지할 수 있다.
•	명확성: URL의 각 구성 요소를 구분하여 관리할 수 있으므로 코드의 유지보수성과 가독성이 향상된다.

앞으로 Java에서 URL을 다루는 코드를 작성할 때는, URI.create()와 toURL()을 활용하는 방식으로 전환하여 개발하도록 해야겠다.

