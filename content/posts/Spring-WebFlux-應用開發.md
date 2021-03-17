---
title: Spring WebFlux 應用開發
tags:
  - Kotlin
  - Java
  - JVM
  - Spring Framework
category:
  - Spring
abbrlink: 49338
date: 2020-09-03 22:23:28
---
# 使用Spring WebFlux保護您的網站應用(驗證篇)

相信瀏覽到這個Topic的各位一定都對 **Reactive** 有相當的認識了，本文章將專注在講解 WebFlux & Reactive Security的應用開發，其餘基礎便不再贅述。

首先我們需要到 [Spring Initializr](https://start.spring.io/) 產生我們的Spring WebFlux Application。

本文章需要的依賴有：**Spring Reactive Web**、**Spring Security**、**Spring Configuration Processor(可選)**
*文章撰寫時 Spring 版本為 2.4.0(M2)，依據版本更新有些地方會稍稍不同。
接下來使用您喜歡的IDE去做開發，本文將使用Intellij IDEA開發

## 前置準備

首先先添加**Jwt**的依賴，我們會於稍後用到。(這部分採用您熟悉的工具也行，也不一定需要用`Jwt`，`PASETO`也是不錯的選擇)

```Gradle
dependencies {
 // jwt dependency
 implementation("com.auth0:java-jwt:3.10.3")
 // other depencies ...
}
```

## 設定網站端點

這部分我們需要配置Spring Security，與以往的Servlet應用不同，不能直接繼承`WebSecurityConfigurerAdapter`去做設定。
首先，在專案底下新增一Package`auth`，把所有驗證、授權邏輯放在這。
在`auth`資料夾底下再新增一個Package`config`，標示其為設定檔所在位置。
在`config`底下新增一類別稱作`AuthSecurityConfig`。
在`AuthSecurityConfig`新增以下程式碼規劃網站授權的Endpoints

```Kotlin
    @Bean
    fun authRoute(http: ServerHttpSecurity): SecurityWebFilterChain {
        return http {
            // csrf 關閉，方便測試
            csrf { disable() }
            // 只篩選/auth開頭的路徑
            securityMatcher(PathPatternParserServerWebExchangeMatcher("/auth/**"))
            // 規劃Endpoints
            authorizeExchange {
                authorize("/auth/login", permitAll)
                authorize("/auth/me", authenticated)
                authorize(anyExchange, authenticated)
            }
            // 關閉 formLogin跟HttpBasic
            formLogin { disable() }
            httpBasic { disable() }
        }
    }
```

規劃完後要思考一件事情，【要如何驗證使用者傳來的資訊?】，這時候`AuthenticationWebFilter`就派上用場了，使用者傳Request進來後會經過層層的過濾器，這個結構稱為FilterChain，那鏈狀結構就有一個特色，就是會依序去篩選Request，最後通過層層過濾器的Request才會到達我們的Controller，`AuthenticationWebFilter`是`SecurityFilterChain`的其中一環，但**未**在 [Spring Security Reference Docs](https://docs.spring.io/spring-security/site/docs/5.3.5.BUILD-SNAPSHOT/reference/html5/#servlet-security-filters) 有詳細介紹。

## 建立AuthenticationManager

在建立`AuthenticationFilter`前，我們需要能夠集中管理授權的管理者，在Spring Security裡被稱為`AuthenticationManager`，同時我們也需要`UserDetailService`才能去初始化`AuthenticationMananger`，這邊採用`MapReactiveUserDetailsService`當作範例，實務開發上請串接MongoDB、MySQL等等的資料庫。

```Kotlin
    @Bean
    @Qualifier("AuthUserService")
    fun UserService(): MapReactiveUserDetailsService {
        val user = User.withDefaultPasswordEncoder()
                .username("user")
                .password("pass")
                .roles("USER")
                .build()
        return MapReactiveUserDetailsService(user)
    }

    @Bean
    fun AuthManager(
            @Qualifier("AuthUserService") service: MapReactiveUserDetailsService
    ): UserDetailsRepositoryReactiveAuthenticationManager {
        return UserDetailsRepositoryReactiveAuthenticationManager(service)
    }
```

## 建立第一個AuthenticationWebFilter

前置準備都弄好了，接下來就要建立第一個`AuthenticationWebFilter`，`AuthenticationWebFilter`已經實現了基礎的驗證，故我們只需要稍作修改即可以使用。
這邊我們會將`DataBuffer`映射為Object，故將使用`ObjectMapper`，請添加依賴：

```Gradle
dependencies {
 // object mapper
 implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
 // other depencies ...
}
```

以下是本人寫的`AuthenticationWebFilter`的**範例程式碼**，不能夠直接使用，也請不要直接使用，會有**安全**上的疑慮。
`JwtService`請自行實現

```Kotlin
    @Qualifier("Authentication")
    @Bean
    fun AuthFilter(manager: UserDetailsRepositoryReactiveAuthenticationManager): AuthenticationWebFilter {
        // 要被驗證的使用者
        var AuthUser: LogInRequest? = null
        val filter = AuthenticationWebFilter(manager)
        filter.setRequiresAuthenticationMatcher { ServerWebExchangeMatchers.pathMatchers(HttpMethod.POST, "/auth/login").matches(it) }
        filter.setServerAuthenticationConverter {exchange ->
            exchange.request
                    .body
                    .next()
                    .switchIfEmpty { Mono.defer { Mono.error<DataBuffer>(RequestBodyIsEmpty()) } }
                    .flatMap { it.ToRead<LogInRequest>() }
                    .filter { it.username != null && it.password != null }
                    .switchIfEmpty { Mono.defer { Mono.error<LogInRequest>(UserInfoError()) } }
                    .map { user ->
                        // 儲存使用者資訊，以便頒發Jwt
                        AuthUser = user
                        UsernamePasswordAuthenticationToken(user.username, user.password)
                    }
        }
        filter.setAuthenticationSuccessHandler { filters, auth ->
            val credential = AuthUser?.let { it.password }
            if (credential != null){
                return@setAuthenticationSuccessHandler filters.exchange.response.let {
                    it.headers.contentType = MediaType.APPLICATION_JSON
                    it.statusCode = HttpStatus.OK
                    it.writeJson(LogInResponse(
                            token = "${JwtService.prefix} ${JwtService.encrypt(auth.name, credential, auth.authorities)}",
                            expiredAt = JwtService.expireTime().format()
                    ))
                }
            }
            else {
                return@setAuthenticationSuccessHandler filters.exchange.response.let {
                    it.headers.contentType = MediaType.APPLICATION_JSON
                    it.statusCode = HttpStatus.UNAUTHORIZED
                    it.writeJson(FailResponse(
                        message = "Illegal Arg."
                    ))
                }
            }
        }
        filter.setAuthenticationFailureHandler { filters, exception ->
            AuthUser = null
            return@setAuthenticationFailureHandler filters.exchange.response.let {
                it.statusCode = HttpStatus.UNAUTHORIZED
                it.headers.contentType = MediaType.APPLICATION_JSON
                it.writeJson(FailResponse(message = exception.message ?: "null"))
            }
        }
        return filter
    }
```

## 加入Filter至SecurityConfig

我們剛才完成了產生Jwt的Filter，現在則是需要加入Filter至SecurityConfig讓其成為FilterChain的一環，這裡因為我們後面還需要有驗證Jwt的Filter，所以將這個過濾器的次序設為`HTTP_BASIC`(比`AUTHENTICATION`前面)

```Kotlin
    @Bean
    fun authRoute(http: ServerHttpSecurity,
                  @Qualifier("Authentication") AuthFilter: AuthenticationWebFilter
    ): SecurityWebFilterChain {
        return http {
            // csrf 關閉，方便測試
            csrf { disable() }
            // 只篩選/auth開頭的路徑
            securityMatcher(PathPatternParserServerWebExchangeMatcher("/auth/**"))
            // 規劃Endpoints
            authorizeExchange {
                authorize("/auth/login", permitAll)
                authorize("/auth/me", authenticated)
                authorize(anyExchange, authenticated)
            }
            // 關閉 formLogin跟HttpBasic
            formLogin { disable() }
            httpBasic { disable() }
            addFilterAt(AuthFilter, SecurityWebFiltersOrder.HTTP_BASIC)
        }
    }
```

## 建立驗證Jwt的Filter

與前面大同小異，故直接貼上範例代碼

```Kotlin
    @Qualifier("Validation")
    @Bean
    fun ValidFilter(manager: UserDetailsRepositoryReactiveAuthenticationManager) : AuthenticationWebFilter{
        val filter = AuthenticationWebFilter(manager)
        filter.setServerAuthenticationConverter { exchange ->
            Mono.just(exchange.request)
                    .flatMap { JwtService.extract(it) }
                    .flatMap { JwtService.parse(it) }
                    .flatMap { JwtService.decrypt(it) }
                    .flatMap { UsernamePasswordAuthBuilder.create(it) }
                    .onErrorResume { exception ->
                        log.info("Jwt Service Authentication failed...")
                        Mono.empty<Authentication>()
                    }
        }
        return filter
    }
```

一樣要將Filter加入至SecurityConfig

```Kotlin
    @Bean
    fun authRoute(http: ServerHttpSecurity,
                  @Qualifier("Authentication") AuthFilter: AuthenticationWebFilter,
                  @Qualifier("Validation") ValidFilter: AuthenticationWebFilter
    ): SecurityWebFilterChain {
        return http {
            // csrf 關閉，方便測試
            csrf { disable() }
            // 只篩選/auth開頭的路徑
            securityMatcher(PathPatternParserServerWebExchangeMatcher("/auth/**"))
            // 規劃Endpoints
            authorizeExchange {
                authorize("/auth/login", permitAll)
                authorize("/auth/me", authenticated)
                authorize(anyExchange, authenticated)
            }
            // 關閉 formLogin跟HttpBasic
            formLogin { disable() }
            httpBasic { disable() }
            addFilterAt(AuthFilter, SecurityWebFiltersOrder.HTTP_BASIC)
            addFilterAt(ValidFilter, SecurityWebFiltersOrder.AUTHENTICATION)
        }
    }
```

最後做一下單元測試就好了，logout的部分找時間再做說明，基本概念就是讓Jwt Token被invalidate掉，但這樣就要牽涉到資料庫了。
