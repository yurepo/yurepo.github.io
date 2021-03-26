---
title: "Hugo 添加 Google Analytics 4 筆記"
date: 2021-03-26T23:35:19+08:00
draft: false
tags:
  - hugo
categories:
  - hugo
---

# Hugo 添加 Google Analytics 4 筆記

## 前言

最近[HUGO](https://github.com/gohugoio/hugo/releases/tag/v0.82.0)更新了0.82.0版本，此版本對比上一版本改進不大，對我來說，最有感的應該是增加了內置[Google Analytics 4](https://support.google.com/analytics/answer/10089681?hl=zh-Hant)的支持，也就是可以使用`G-`開頭的評估ID而不是`UA-`開頭的追蹤ID，那首先就來回顧一下Universal Analytics(GA3)與Google Analytics 4。

## GA4 x GA3

### GA3 (Universal Analytics)

UA於2012年10月發布，在上一代GA中，主要以網站會話(session)以及網站頁面為主，資料的來源主要從三個管道收集

1. HTTP請求
2. 瀏覽器/系統資訊
3. 第一方Cookie

其中，我們最容易見到的是帶有`utm`開頭參數的連結，範例如下

範例：
`http://example.com/?utm_source=active%20users&utm_medium=email&utm_campaign=feature...`

這種分享連結後方都會添加一些`utm`開頭之參數，這個`utm`參數會記錄其來源、媒介、名稱，等到使用者造訪網站時，在網站上嵌入的UA會去收集這些參數並發送至Google Analytics，讓網站管理者檢視、分析成效。

上方的範例僅限於使用者是被哪個社群網站、哪個廣告以及從哪個來源(e.g. email、app)吸引而進入網站，那如何去收集使用者在網站內部的活動呢?

UA擁有許多不同匹配去收集使用者在網站內部的活動，這邊介紹最常見的三種：

1. 網頁瀏覽匹配: 使用者載入嵌有UA的頁面時便會觸發此匹配，是最常觸發的操作。
2. 事件匹配: 追蹤使用者在網站上特定元素的每次互動，例如：開啟的網址、播放的影片等。
3. 交易匹配: 傳送電子商務購買的相關資料，例如：售出的產品、交易ID、庫存計量單位(SKU)等等。

除上述三種之外也有許多其他匹配，像是社交匹配、網頁操作時間匹配等

### GA4 (Google Analytics 4)

GA4於2020年10月16日發布，在這一代GA中，主要採用以事件(event-based)的方式去收集資料，相較於上一代GA3，GA4提供了更彈性、更智慧、跨平台的數據蒐集方法。

GA4與GA3同樣都使用`gtag.js`向Google Analytics發送事件數據。

在以往的GA3，如果想要評估App端上的使用數據，必須使用[Google Analytics for Firebase](https://firebase.google.com/products/analytics)或是[Google Analytics APP view](https://support.google.com/analytics/answer/6317479?hl=en&ref_topic=2587085)建立不同的GA資源(Property)，想要結合網頁端及App端的使用數據相對來說是較不容易的。

GA4 整合了 Web 端及 APP 端的資料，並可以將其結合在一起進行分析，也可以單獨蒐集網站上的資料。

GA4默認提供了六種增強性評估

1. 網頁瀏覽
2. 捲動
3. 外連點擊
4. 站內搜尋
5. 影片參與
6. 檔案下載

若要詳細了解GA4的功能及比較，可至[【一表看懂】新版 GA4 與舊版 GA 差在哪裡？新舊版本功能比較懶人包！](https://www.turingdigital.com.tw/blog/app-web-property-comparison)

**TL;DR**：新版GA4相較於上一代GA3更彈性、更智慧(後臺方面)、跨平台，但GA3上有的功能並不一定在GA4上可以找到替代品，若是打算由GA3遷移到GA4，可保留GA3和GA4兩者。不過對於Hugo使用者來說，不管是選擇GA3或是GA4都應該可以達成目的。

## 如何在 Hugo 設定 GA4

接下來進入到正題，如何在新版Hugo設定GA4呢?

其實在新版Hugo設定GA4與設定UA並無不同

### 第一步，設定`config.yml`

首先在部落格根目錄的`config.yml`必須加上

```yaml
googleAnalytics: <GA4_評估_ID>
```

(取得GA4評估ID請至[Google Analytics](https://analytics.google.com/)，這邊並不贅述)

### 第二步，向主題上新增GA4模板

並且在您主題的模板(template)上新增

```html
<!-- Add GA4 support -->
{{ template "_internal/google_analytics.html" . }}
```

(記得這段應在HTML中的`<head>`區段中增加)，以我目前使用的主題`PaperMod`為範例，應在`...\themes\PaperMod\layouts\partials\head.html`中添加)

## 參考資料

[Google Analytics - Internal Templates](https://gohugo.io/templates/internal#google-analytics)

[【一表看懂】新版 GA4 與舊版 GA 差在哪裡？新舊版本功能比較懶人包！](https://www.turingdigital.com.tw/blog/app-web-property-comparison)

[什麼是UTM？如何使用UTM追蹤成效、數據與流量？](https://www.awoo.com.tw/blog/utm/)

[透過自訂網址收集廣告活動資料](https://support.google.com/analytics/answer/1033863?hl=zh-Hant)

[跟踪代码概览](https://developers.google.com/analytics/resources/concepts/gaConceptsTrackingOverview)

[How Google Analytics collects data (5:39)](https://www.youtube.com/watch?v=lpMmIPWuKTk)
