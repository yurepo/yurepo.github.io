---
title: Spring Security 02 - Security Context? 從哪來?
abbrlink: 27387
date: 2020-10-08 19:13:40
tags:
  - Spring Security
  - Reactive
categories:
  - Spring
---
**上回**我們說了`SecurityContextHolder`主要在做什麼，這回主要討論的是，`SecurityContext`到底是何方神聖，它從何而來。

`SecurityContext`它作為 Spring Security 核心中的一部分，它的作用可說是非常重要的。我們來看看`SecurityContext`的源碼。

```Java
public interface SecurityContext extends Serializable {
	/**
	 * 獲取當前已驗證的身分或驗證請求令牌
	 *
	 * @return the <code>Authentication</code> or <code>null</code> if no authentication
	 * information is available
	 */
	Authentication getAuthentication();

	/**
	 * 改變目前已驗證的身分或刪除驗證資料
	 *
	 * @param authentication the new <code>Authentication</code> token, or
	 * <code>null</code> if no further authentication information should be stored
	 */
	void setAuthentication(Authentication authentication);
}
```
可以看到`SecurityContext`主要是在管理`Authentication`的物件，那這個`Authentication`主要是存放當前使用者的身分(廢話)、存放是否已驗證以及取得目前使用者擁有的權限。

那 `SecurityContext` 從何而來呢? 同個包內的`SecurityContextImpl`就是它的實現，裡面都是很基本的邏輯，這次就不放了，可從[官方GitHub](https://github.com/spring-projects/spring-security/)找到相關源代碼。

那如何在不同request之間保存`SecurityContext`呢?
在Spring Security主要有兩個在不同request之間保存`SecurityContext`的策略，這邊簡單介紹一下，兩者都是`ServerSecurityContextRepository`的實現。
1. `NoOpServerSecurityContextRepository`: 很廢，當你叫它保存的時候它不會理你，左耳進右耳出，然後你問他那個東西在哪，它會直接說"我不知道，你有跟我講過嗎?"，通常用於無狀態(Stateless)驗證，例如：httpBasic。
2. `WebSessionServerSecurityContextRepository`: 好學生，你叫它保存的時候，它會幫你做三件事，**1.** 取得當前Session -> **2.** 幫你把東西保存至Session -> **3.** 順便幫你把SessionId換成新的。通常用於formLogin。

雖然第二種方法非常理想，可是要知道儲存是非常昂貴的一件事，所以在`InMemoryWebSessionStore`有定義說，Session預設就只能有10000個，再多它就會跟你鬧脾氣(丟錯誤)。
當然你可以改變預設值使其達到你預估的數量，但最好的解決方法是自定義你的`WebSessionStore`用以支持更多Session。(在`DefaultWebSessionManager`可設置您的自定義`WebSessionStore`)。

這回我們簡單介紹了`SecurityContext`以及提了一些相關的API，下回預計介紹各種`Filters`。
