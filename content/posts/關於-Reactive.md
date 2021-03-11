---
title: Spring Security 01 - 關於 ReactiveSecurityContextHolder 的一兩件事 
abbrlink: 18626
date: 2020-10-06 22:28:18
tags:
  - Spring Security
  - Reactive
categories:
  - Spring
---
Spring Security對於寫過Spring Boot的人應該是再熟悉不過了，這篇文主要紀錄我對於 `ReactiveSecurityContextHolder`的理解，原始碼版本為**5.4.0-RC1**。

在聊原始碼前，我想先聊一下Context到底是什麼。

**Context**，中文譯作「上下文」，我對於上下文的理解就是**物件作用的環境**。

打個比方，假設現在有一個物件叫做`Weather`，而假設`Weather`物件會有下雨、晴朗這兩種狀態，且`EarthContext`封裝其物件或者是強制其改變為某種狀態時，則可以說`Weather`的上下文是`EarthContext`。

對於Spring Security來說，`SecurityContext`就是整個Spring Security應用的上下文。

而`SecurityContextHolder`就是單純保存這個上下文而存在的。
在傳統Servlet應用`SecurityContextHolder`是存在多種保存上下文的策略，比方說`GlobalSecurityContextHolderStrategy`、`InheritableThreadLocalSecurityContextHolderStrategy`跟`ThreadLocalSecurityContextHolderStrategy`，但在`ReactiveSecurityContextHolder`中，並沒有多種策略去保存`SecurityContext`，唯一保存上下文的方法就是透過Reactor的[上下文](https://projectreactor.io/docs/core/release/reference/index.html#context)。

以下為個人部分翻譯的源碼：

```Java
public class ReactiveSecurityContextHolder {
	private static final Class<?> SECURITY_CONTEXT_KEY = SecurityContext.class;

	/**
	 * 從 Reactor {@link Context} 取得 {@code Mono<SecurityContext>} 
	 * @return 回傳 {@code Mono<SecurityContext>}
	 */
	public static Mono<SecurityContext> getContext() {
        // 從Reactor的上下文中取得SecurityContext
		return Mono.subscriberContext()
			.filter( c -> c.hasKey(SECURITY_CONTEXT_KEY))
			.flatMap( c-> c.<Mono<SecurityContext>>get(SECURITY_CONTEXT_KEY));
	}

	/**
	 * 從 Reactor {@link Context} 清除 {@code Mono<SecurityContext>}
	 * @return 清除Reactor上下文，並回傳一個 Mono<Void>，若清除失敗，則報錯。
	 */
	public static Function<Context, Context> clearContext() {
        // 從Reactor的上下文中刪除SecurityContext
		return context -> context.delete(SECURITY_CONTEXT_KEY);
	}

	/**
	 * 創建含有 {@code Mono<SecurityContext>} 的 Reactor {@link Context}
	 * 可與其他 {@link Context} 合併 
	 * @param securityContext the {@code Mono<SecurityContext>} to set in the returned
	 * Reactor {@link Context}
	 * @return a Reactor {@link Context} that contains the {@code Mono<SecurityContext>}
	 */
	public static Context withSecurityContext(Mono<? extends SecurityContext> securityContext) {
		return Context.of(SECURITY_CONTEXT_KEY, securityContext);
	}

	/**
	 * A shortcut for {@link #withSecurityContext(Mono)}
	 * @param authentication the {@link Authentication} to be used
	 * @return a Reactor {@link Context} that contains the {@code Mono<SecurityContext>}
	 */
	public static Context withAuthentication(Authentication authentication) {
		return withSecurityContext(Mono.just(new SecurityContextImpl(authentication)));
	}
}
```

了解了 `ReactiveSecurityContextHolder` 主要在幹嘛，下一篇預計會介紹 SecurityContext 從何而來。

