---
title: JwtAuthenticationFilter를 이용한 Spring Security JWT 인증 필터 설정 방법
description: >-
  Jwt token 검증 custom 필터 세팅
author: author_js
date: 2025-03-25 01:49:00 +0900
categories: [JAVA, Spring Boot]
tags: [java, spring, boot, 자바, 스프링부트, 스프링, security, JWT, filter]
pin: true
---
Spring Boot와 Spring Security를 사용하여 JWT 토큰을 서버에서 인증하는 방법에 대해서 알아보도록 하겠습니다.

### 1. JwtAuthenticationFilter 이해하기

`JwtAuthenticationFilter`는 JWT 토큰을 검증하여 사용자를 인증하는 역할을 하는 필터입니다. 이 필터는 모든 요청을 가로채어, 요청 헤더에 JWT가 포함되었는지 확인하고 토큰이 유효한 경우 사용자를 인증합니다.

### 2. JwtAuthenticationFilter 클래스 구성

`JwtAuthenticationFilter`는 다음과 같은 주요 작업을 수행합니다:

- HTTP 요청에서 JWT 추출

- JWT 유효성 검증

- 사용자 정보 로드 및 인증 처리


### 3. SecurityConfig 설정

Spring Security 설정에서 `JwtAuthenticationFilter`를 등록합니다.

```
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    public static final String[] AUTH_WHITELIST = {
        "/auth/**",
        "/public/**"
    };

    private final JwtTokenProvider jwtTokenProvider;
    private final UserDetailsService userDetailsService;

    public SecurityConfig(JwtTokenProvider jwtTokenProvider, UserDetailsService userDetailsService) {
        this.jwtTokenProvider = jwtTokenProvider;
        this.userDetailsService = userDetailsService;
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf(AbstractHttpConfigurer::disable)
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(AUTH_WHITELIST).permitAll()
                .anyRequest().authenticated()
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .addFilterBefore(new JwtAuthenticationFilter(jwtTokenProvider, userDetailsService), UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

### 4. JwtTokenProvider 작성

토큰의 생성, 검증, 사용자 이름 추출 등을 담당하는 `JwtTokenProvider`를 작성합니다.

### 5. 실행 및 테스트

이제 서버를 실행하고, 프론트 엔드에서 API 요청을 보낼때 JWT 토큰을 담아 보내 정상적으로 토큰을 검증하는지 확인을 하면 됩니다. Request Header에 아래와 같이 토큰을 넣어 보내면 됩니다. 

```
Authorization: Bearer {토큰값}
```

### 결론

이렇게 설정하면 서버에서 JWT 토큰을 검증에 성공한 요청만 컨트롤러에 연결이 되게 됩니다. 검증이 필요없는 요청들은 AUTH_WHITELIST에 세팅해서 검증없이 요청을 승인하도록 하면 됩니다. 