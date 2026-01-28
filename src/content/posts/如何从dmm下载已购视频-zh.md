---
title: 如何从dmm下载已购视频
published: 2026-01-15T03:13:10.513Z
description: ''
updated: ''
tags:
  - 创造与分享
draft: false
pin: 0
toc: true
lang: ''
abbrlink: ''
---

在 DMM/Fanza 购买影片后，如何下载 MP4 格式到本地？在官方提供的播放渠道中，并没有包含此一选项

![](https://image.906732.xyz/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-01-15%20122923.png)

所谓的下载也只是下载一个加密的 dcv 文件，需要配合专用的播放器才能实现播放

![](https://image.906732.xyz/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-01-15%20123431.png)

那么如何才能下载到不加密的视频文件呢？还是需要以**在线播放**为突破口，只要能在电脑上播放，就一定有下载的方法，通过 **cat-catch** 我们可以发现，页面加载了这些资源：

![](https://image.906732.xyz/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-01-15%20140413.png)

但如果你想直接下载，得到的文件是无法打开的：

![](https://image.906732.xyz/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-01-15%20140548.png)

因为它使用了 **DASH (Dynamic Adaptive Streaming over HTTP)** 技术。它不直接提供一个完整的 MP4 文件，而是通过 **MPD (Media Presentation Description)** 文件作为索引，将视频和音频切成无数个 `.m4s` 或 `.ts` 小片进行传输 。所以核心是抓取 **MPD链接**

![](https://image.906732.xyz/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-01-15%20140946%20(1).png)

但是 **DASH** 只是传输协议，并不负责加锁，因此我们要进一步检查，它的加锁方式是什么，F12打开控制台，发现“license”相关的请求（https://mlic.dmm.co.jp/drm/clearkey/license）：

![](https://image.906732.xyz/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-01-15%20142459.png)

**这是 ClearKey 的标准格式**：JWK (JSON Web Key)。它非常直观，直接把密钥 ID (`kid`) 和密钥 (`k`) 放在数组里：

- `kid` 是密钥的 ID，用于标识这段密钥对应哪个流，将这个 base64 转为 hex：GcbW7lIkNvGzbMd1M5GmfQ -> 19c6d6ee522436f1b36cc7753391a67d

- `k` 字符串非常长（约 250+ 字符），在标准的 ClearKey 中，密钥通常是 16 字节（32 位 Hex），所以这个这么长的 `k` 值通常说明 DMM 在响应中加密了真正的 Key，或者这是一个经过包装的 Token。但对于 DMM 这种大型平台，通常需要特定的解密脚本或工具（如专用的浏览器扩展/脚本）来处理这个长字符串以获得原始的 32 位 Hex 密钥，因此**密钥很难通过手动方式尝试破解，最好的方法还是“内存抓取”**

ClearKey 机制：顾名思义，它是“明文密钥”。虽然在传输过程中 `k` 值可能被包装得很复杂（上面到的超长字符串），但 W3C 标准规定：**浏览器必须能将这些数据还原成标准的 AES-128 密钥，并将其交给浏览器的内容解密模块 (CDM)**

所以，**突破点**就在：ClearKey 的解密逻辑通常是在浏览器的用户态进程中完成的。为了让解密引擎工作，系统必须在内存的某个时刻存在一份**完全解开的、16 字节的原始密钥**

安装浏览器插件 **EME Logger**，刷新网站，就能得到对应的日志：

![](https://image.906732.xyz/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-01-15%20144332.png)

另存为本地文件，打开并搜索`MediaKeySession.update`

![](https://image.906732.xyz/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-01-15%20144709%20(1).png)

这些 `0x7b`, `0x22` 等十六进制数值，其实就是计算机中字符的 **ASCII 码**。我通过转换发现，这段数据其实是一个 **JSON 字符串**

- `0x7b` = `{`

- `0x22` = `"`

- `0x6b, 0x65, 0x79, 0x73` = `keys`

将整个数据块转换后，就得到了标准的 **JWK (JSON Web Key)** 结构：

```
{
    "keys": [
        {
            "kty": "oct",
            "k": "t55KG1DxXXXXXXXXXXXXXX",
            "kid": "GcbW7lIkNvGzbMd1M5XXXX"
        }
    ]
}
```

再次进行Base64URL 到 Hex 的转换：

**KID**: `GcbW7lIkNvGzbMd1M5GmfQ` $\rightarrow$ `19c6d6ee522436f1b36cc7753391a67d`

**Key**: `t55KG1DxXXXXXXXXXXXXXX` $\rightarrow$ `b79e4a1b50f15d75d75d75d75d75d75d`

这样我们就可以使用 **N_m3u8DL-RE** 下载文件：

![](https://image.906732.xyz/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-01-15%20150212.png)

下载完成后，自动合并视频和音频，得到最终文件：

![](https://image.906732.xyz/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-01-15%20150513.png)

最后，我将这一整个流程，整合为了一个 Chrome 插件：[项目地址](https://github.com/ianho7/dmm-download-helper?tab=readme-ov-file)，里面包含详细的介绍、[下载地址](https://github.com/ianho7/dmm-download-helper/releases)，如果对你有帮助，也欢迎给我一个免费的Star，演示视频如下：[![视频标题](https://image.906732.xyz/output.gif)](https://image.906732.xyz/output.webm)
