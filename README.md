# 微信 Windows 视频号启动约 15 秒：重复校验前端资源与 `max_age` 拼写问题 / WeChat Channels: ~15s startup, repeated frontend validation and `max_age` typo

## 摘要

在 Windows 微信中打开视频号卡片时，视频加载反复需要等待约 15 秒才有明显进展。实测等待时间与一系列前端资源连接重合。检查的 77 个视频号前端缓存资源均声明 `Cache-Control: max_age=31536000`，而标准指令应为 `max-age`。

缓存指令的拼写问题已验证。现有证据强烈支持重复校验是启动延迟的影响因素，但尚未在微信内进行修正响应头前后的 A/B 对照实验。本报告不能证明全部延迟都由这一拼写问题造成。

## 环境与验证范围

- 调查日期：2026 年 10 月 24 日，America/New_York。
- Windows 11 25H2，build26200.9457。
- Weixin Windows4.2.0.8；WMPF runtime26042。
- 实测范围：一个 Windows 桌面实例；未对其他运行配置逐一进行独立计时。
- 用户反馈的对比：桌面端需 15s+，同一网络下手机端 <1s。未对手机端进行独立计时观测。
- 抓包时涉及的资源节点：`res.wx.qq.com`，IP43.146.26.163。结论限于当时观察到的资源、日期和分发路径；其他 CDN 区域及版本尚未验证。

## 实际执行的复现步骤

1. 保持微信运行，打开包含视频号分享卡片的聊天。
2. 打开一张卡片，等待其查看器完成加载。
3. 在聊天中打开另一张卡片，创建另一个视频号页面或标签页。
4. 将该启动过程与在已加载的查看器内滚动切换视频进行比较。

预期行为：打开视频后应及时进入加载流程；缓存策略允许时，应直接复用本地仍然有效的版本化前端资源。

实际行为：即使微信和浏览器宿主进程已经运行，打开另一张卡片仍会再次经历较长的前端加载等待。在实测对比中，已加载查看器内的切换明显更快。

## 实测时间线

以下时间均相对于各自操作，不是精确的首帧时间测量。

| 观测项 | 打开另一张聊天卡片 | 在已加载的查看器内滚动切换 |
| --- | ---: | ---: |
| 新建资源 CDN TCP 连接 | 51 个，启动时间覆盖 +1.06s 至 +14.57s | 在抓包覆盖范围内，随后 10s 没有新建此类连接 |
| 通过媒体域名 `finder.video.qq.com` 识别的新连接 | +15.65s | +0.75s |
| 截图检查时已能看到播放 | +19.38s | +3.35s |

延迟后的媒体连接建立不久，视频相关连接便传输了数 MB 数据。这表明页面及前端启动是实测中的主要等待阶段，而非视频传输始终缓慢。已有连接或后台播放可能重叠；加密数据包无法逐一识别所有 HTTP 请求。

## 缓存策略证据

公开资源示例：

https://res.wx.qq.com/t/wx_fed/finder/web/web-finder/res/js/IconComplaint.publishCUcT-3OJXo.js

观测到的部分响应头：

```http
HTTP/1.1 304 Not Modified
Etag: "71ee8d467fafcfba96d3a2132cba7b47"
Date: Thu, 22 Oct 2026 06:47:01 GMT
Connection: close
Cache-Control: max_age=31536000
```

这个 304 来自对上述同一资源单独发起的 curl 条件 GET 请求，并非解密微信流量得到。该请求约 0.84s 完成；另一个接受压缩响应的条件 GET 请求耗时 1.34s。无条件 GET 请求在 1.42s 内返回 200。

检查的 77 个视频号缓存资源全部带有相同的下划线指令。其中 45 个资源的请求时间戳位于第二次播放测试期间，同时仍保留更早的原始响应时间戳。缓存检查采用只读、按格式解析的方式，但不是跨文件的原子快照；未独立验证缓存索引成员关系及条目自身哈希。

示例中的 Date 与 Last-Modified 相差 538 秒。按 Chromium 的 10% 启发式计算，有效期约为 53.8 秒，而接收时的表观年龄约为 239,423 秒。该条目中没有发现 Expires 响应头。`max_age` 无法表达预期的一年缓存有效期。

[HTTP 缓存标准](https://www.rfc-editor.org/rfc/rfc9111.html#section-5.2.3)要求缓存忽略无法识别的指令。[Chromium 实现](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/net/http/http_response_headers.cc)识别的是 `max-age`，缺少显式有效期时会采用启发式计算。

### 用于独立复现的公开资源检查

以下 Windows curl 命令无需微信账号或私人视频链接即可复现响应头检查。调查日期之后，版本化资源和 ETag 可能发生变化或失效。

```powershell
curl.exe --max-time 15 --dump-header - --output NUL "https://res.wx.qq.com/t/wx_fed/finder/web/web-finder/res/js/IconComplaint.publishCUcT-3OJXo.js"
curl.exe --max-time 15 --compressed --header 'If-None-Match: "71ee8d467fafcfba96d3a2132cba7b47"' --dump-header - --output NUL "https://res.wx.qq.com/t/wx_fed/finder/web/web-finder/res/js/IconComplaint.publishCUcT-3OJXo.js"
```

## 推断与待验证事项

现有证据支持：前端资源加载及校验过程拖延了新页面的视频播放。缺失预期的缓存有效期、条件响应关闭连接，以及模块化资源依赖，都可能放大这一开销。

以下事项尚未验证：

- 每个微信请求的准确 HTTP 状态、响应头和缓存模式；本次没有解密 TLS。
- 修正静态资源的缓存有效期指令后，是否能消除微信内实测的延迟。
- 缓存校验、模块加载依赖、服务器延迟和原生桥接初始化各自贡献的时间。
- 其他 Windows 运行配置与手机端的独立复现，以及其他 CDN 节点、客户端版本的行为。
- 先前正常工作的版本或回归引入点：`NOT_ENOUGH_INFO`；未进行受控版本对比。

## 建议上游验证

1. 将适用的版本化静态资源缓存指令修正为 `max-age=31536000`，再重复冷缓存、热缓存下的页面打开计时。
2. 检查条件响应的连接复用方式和前端依赖加载顺序。
3. 对比打开另一张聊天卡片与在已有查看器中切换视频。

调查期间没有改写缓存或响应头，也没有调整客户端设置。在一次实测中，保留查看器并在其中切换视频避开了长时间等待；这可能受益于预取，不构成普遍的性能保证。

这是一份基于特定测试环境的调查记录；其他环境仍需独立复现。

<details>
<summary>English version</summary>

## Summary

Opening a WeChat Channels (视频号) card on Windows repeatedly takes roughly 15 seconds before video loading progresses. The measured delay coincides with a sequence of frontend-resource connections. All 77 examined cached Channels frontend resources advertise `Cache-Control: max_age=31536000`; the standard directive is `max-age`.

The misspelled directive is verified. Repeated validation is strongly supported as a contributor to startup latency, but a corrected-header A/B test inside WeChat has not been performed. This report does not establish that the typo alone accounts for the entire delay.

## Environment and scope

- Investigation date: October 24, 2026, America/New_York.
- Windows 11 25H2, build26200.9457.
- Weixin Windows4.2.0.8; WMPF runtime26042.
- Measurement coverage: one Windows desktop instance; other runtime configurations were not independently timed.
- User-reported comparison: 15s+ on desktop versus <1s on a phone using the same network. Mobile timings were not independently instrumented.
- Relevant resource edge during the packet capture: `res.wx.qq.com`, IP43.146.26.163. Findings are specific to the observed resources, date and delivery path; other CDN regions/versions are unverified.

## Reproduction performed

1. Start with WeChat running and a chat containing shared Channels cards.
2. Open one card and allow its viewer to load.
3. Open a different card from the chat, creating another Channels page/tab.
4. Compare that startup with scrolling to another video inside an already-loaded viewer.

Expected behavior: opening a video should progress promptly, with reusable versioned frontend assets served from a fresh local cache when policy allows.

Observed: opening another card recreates a long frontend-loading interval even though the application and browser host are already running. Switching within the loaded viewer was substantially faster in the measured comparison.

## Measured timeline

Times are relative to the respective action, not precise first-frame measurements.

| Observation | Open a different chat card | Scroll within loaded viewer |
| --- | ---: | ---: |
| New resource-CDN TCP connections | 51 starts spanning +1.06s to +14.57s | None started in the following10s within capture coverage |
| New connection identified by media hostname `finder.video.qq.com` | +15.65s | +0.75s |
| Playback visibly running at screenshot check | +19.38s | +3.35s |

Video-related connections transferred megabytes shortly after the delayed media connection began. This points to page/frontend startup as the dominant measured wait rather than uniformly slow video transfer. Existing connections/background playback may overlap; encrypted packets do not identify every individual HTTP request.

## Cache-policy evidence

Example public resource:

https://res.wx.qq.com/t/wx_fed/finder/web/web-finder/res/js/IconComplaint.publishCUcT-3OJXo.js

Selected observed headers:

```http
HTTP/1.1 304 Not Modified
Etag: "71ee8d467fafcfba96d3a2132cba7b47"
Date: Thu, 22 Oct 2026 06:47:01 GMT
Connection: close
Cache-Control: max_age=31536000
```

This 304 was observed in an independent conditional curl GET to that exact resource, not decrypted from WeChat traffic. It completed in approximately0.84s; another compressed conditional GET completed in1.34s. An unconditional GET returned200 in1.42s.

The same underscore directive appeared on all77 examined cached Channels resources. Forty-five had request timestamps during the second playback attempt while retaining older original-response timestamps. Cache inspection was read-only and format-aware, but not an atomic snapshot across files; cache index membership and entry selfhashes were not independently verified.

For the example, Date and Last-Modified differ by538seconds. Chromium's 10% heuristic would yield about53.8seconds of freshness, while apparent age at receipt was about239,423seconds. No Expires header was found for that entry. The intended one-year lifetime is not conveyed by `max_age`.

The [HTTP caching standard](https://www.rfc-editor.org/rfc/rfc9111.html#section-5.2.3) requires caches to ignore unrecognized directives. [Chromium's implementation](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/net/http/http_response_headers.cc) recognizes `max-age` and applies heuristic freshness when explicit lifetime is unavailable.

### Public-resource check for independent reproduction

These Windows curl commands reproduce the header checks without a WeChat account or private video URL. Versioned resources and ETags may change or disappear after the investigation date.

```powershell
curl.exe --max-time 15 --dump-header - --output NUL "https://res.wx.qq.com/t/wx_fed/finder/web/web-finder/res/js/IconComplaint.publishCUcT-3OJXo.js"
curl.exe --max-time 15 --compressed --header 'If-None-Match: "71ee8d467fafcfba96d3a2132cba7b47"' --dump-header - --output NUL "https://res.wx.qq.com/t/wx_fed/finder/web/web-finder/res/js/IconComplaint.publishCUcT-3OJXo.js"
```

## Interpretation and remaining checks

The evidence supports a frontend-resource loading/validation sequence that delays new-page playback. The missing intended cache freshness, conditional-response connection closure and modular resource dependencies plausibly amplify that cost.

Still unverified:

- Exact HTTP status/headers/cache mode of each WeChat request; TLS was not decrypted.
- Whether correcting the static-asset freshness directive removes the measured delay inside WeChat.
- Relative contribution of validation, module-loading dependencies, server latency and native bridge initialization.
- Independent reproduction under other Windows runtime configurations and on mobile, and behavior on other CDN edges/client versions.
- Previous known-good version or regression introduction: `NOT_ENOUGH_INFO`; no controlled version comparison was performed.

## Proposed upstream checks

1. Correct the cache directive for appropriate versioned static assets to `max-age=31536000`, then repeat cold/warm page-open measurements.
2. Inspect conditional-response connection reuse and frontend dependency-loading order.
3. Compare another chat-card open with a video change inside an existing viewer.

No cache/header rewrite or client setting change was applied during this investigation. Keeping the viewer open and switching within it avoided the long delay in one measured test; this may benefit from prefetch and is not a universal performance guarantee.

This investigation covers one specific test environment; independent reproduction is needed for other environments.

</details>
