---
title: securityFilterChain 설정
description: >-
  스프링 부트 security config 설정(1)
author: author_js
date: 2023-03-22 23:00:00 +0900
categories: [SpringBoot, Security]
tags: [스프링, 스프링부트, 보안, 필터, spring, springboot, filter, chain, setting, config]
pin: true
---

# 스프링 부트 Security 설정법 (1)
> 스프링 부트의 Security 설정에 대해서 알아보자.

> 나도 정확하겐 모른다. 그냥 내가 하고 있는 설정에 대해서 알아보자!
{: .prompt-tip }

먼저 security config의 security filter chain 코드입니다. 개발을 처음 시작했을때 고생한 결과로 얻은 이 필터체인 코드로 인해서 지금은
프로젝트의 시작을 꽤 빠르게 할 수 있지만, 처음 security config를 작성했을 당시에는 스프링 부트의 버전이 무엇인지, Spring Security 의 버전이
무엇인지 잘 몰라 그냥 최신 버전으로 생성을 했지만, 당시에 스프링 부트 3버전 이후, 그리고 그에따른 스프링 시큐리티 6에 대한 설정 코드를
공유가 잘 되지 않고, 옛날 버전으로 세팅하면서 이해도 되지 않은 코드들이 실행이 안되서 굉장히 고생했었습니다.

그리고 회사 자체가 제가 입사하기 전에는 PHP를 이용한 그누보드라는 프레임워크를 이용해서 웹을 만들었었고, 전임자가 퇴사하면서 제가 입사를 했기 때문에
SpringBoot를 이용해서 개발을 하는 것은 처음이었고, 이전에 교육원에 다닐때는 JSP를 이용한 개발을 배웠지만, 또 Vue3라는 프론트 엔드 프레임 워크를
선택해서 개발을 했기 때문에 더욱 Security를 세팅을 혼자 해내야 했기 때문에 정말 고생했습니다.

시간이 흘러 지금 이 코드들을 보니까 왜 그렇게 고생을 했을까 라는 의문이 들 정도로 상당히 간단하게 만들어져있고, Spring Boot라는 프레임 워크가
얼마나 쉽고 빠르게 개발을 할 수 있게 해주는지를 깨닳을 수 있는 것 같습니다.

## securityFilterChain
```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
            .csrf(AbstractHttpConfigurer::disable)
            .authorizeHttpRequests(authorize -> authorize
                    .requestMatchers(AUTH_WHITELIST).permitAll()
                    .anyRequest().authenticated()
            )
            .exceptionHandling(exception -> exception
                    .authenticationEntryPoint(authenticationEntryPoint)
            )
            .sessionManagement(session -> session
                    .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .cors(withDefaults());

    http.addFilterBefore(authenticationFilter, UsernamePasswordAuthenticationFilter.class);

    return http.build();
}
```

제 Spring Boot Security의 설정 코드입니다. 이 메서드는 간단하게 HTTP 요청이 들어왔을 때 어떤 보안 정책을 적용할지를 정의하는 필터 체인이라고
보면 될 것 같습니다.

`.csrf(AbstractHttpConfigurer::disable)` 이 설정은 먼저 CSRF(Cross-Site-Request Forgery) 보호를 비활성화하는 코드입니다. 저는 JWT
기반 인증을 하는 API 서버로 사용하기 때문에 CSRF 보호를 비활성화해놨습니다.

### HTTP 요청 권한 설정
`.requestMatchers(AUTH_WHITELIST).permitAll()` : AUTH_WHITELIST에 정의된 URL은 권한 인증을 하지 않고, 접근을 허용합니다. 보통 로그인,
회원가입, 정적 리소스 등 인증이 필요하지 않고 공개되어야 하는 엔드포인트를 지정해서 허용합니다.

`.anyRequest().authenticated()` : AUTH_WHITELIST에 정의되지 않은 모든 URL은 인증이 필요하도록 설정합니다.

### 예외 처리 설정
```java
.exceptionHandling(exception -> exception
        .authenticationEntryPoint(authenticationEntryPoint)
)
```
인증이 필요한 요청이지만 인증이 되지 않은 사용자가 접근할 경우 설정해놓은 Custom authentication entry point를 통해서 응답을 반환합니다.

### 세션 관리
```java
.sessionManagement(session -> session
        .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
)
```

`SessionCreationPolicy.STATELESS` : 이 어플리케이션의 상태를 stateless 방식으로 동작하도록 지정하는 설정입니다. JWT를 사용한 토큰 기반
인증이기 때문에 서버에서 세션을 유지 하지 않고, 매 요청마다 토큰을 확인하여 인증하는 방식입니다.

### CORS 설정
`.cors(withDefaults());` : CORS(Cross-Origin Resource Sharing) 클라이언트가 다른 출처에서 요청할 때, 허용된 출처만 접근하도록 제어하는
설정입니다. 여기서는 기본값으로 CORS를 활성화 했는데, CorsConfigurationSource 빈을 통해서 설정을 했습니다.

### 커스텀 필터 추가
`http.addFilterBefore(authenticationFilter, UsernamePasswordAuthenticationFilter.class);` : JWT 토큰 인증을 하는 커스텀 인증
로직을 수행하는 필터입니다. `addFilterBefore`를 통해서 커스텀 인증 방식이 우선 처리되어서 이후 필터들이 인증된 사용자의 요청만 처리하도록 합니다.

## 결론
저의 Security filter의 설정은 이렇게 구성되어 있습니다. 아직 많은 배움이 더 필요하기에 분명 더 많은 설정들을 해야할 것입니다. 계속해서 발전해
나가도록 노력해야할 것 같습니다. 개발을 하면서 굉장히 시간이 부족하다는 생각을 많이합니다. 배울게 너무 많고, 해야할 것도 너무 많습니다. 꾸준히 나아갑시다!