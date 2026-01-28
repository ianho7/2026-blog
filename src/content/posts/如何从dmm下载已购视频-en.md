---
title: How to Download Purchased Videos from DMM
published: 2026-01-15T03:13:10.513Z
description: ''
updated: ''
tags:
  - Creation and Sharing
draft: false
pin: 0
toc: true
lang: 'en'
abbrlink: 'how-to-download-dmm-video'
---

How to download MP4 format videos locally after purchasing from DMM/Fanza? The official playback options do not include this feature.

![](https://image.906732.xyz/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-01-15%20122923.png)

The so-called download is merely an encrypted .dcv file, which requires a dedicated player for playback.

![](https://image.906732.xyz/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-01-15%20123431.png)

So, how can one download a non-encrypted video file? The key still lies in **online playback**. As long as it can be played on a computer, there must be a way to download it. Through **cat-catch**, we can discover that the page loads these resources:

![](https://image.906732.xyz/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-01-15%20140413.png)

However, if you try to download directly, the resulting file cannot be opened:

![](https://image.906732.xyz/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-01-15%20140548.png)

This is because it uses **DASH (Dynamic Adaptive Streaming over HTTP)** technology. It does not provide a complete MP4 file directly. Instead, it uses an **MPD (Media Presentation Description)** file as an index, splitting the video and audio into numerous small `.m4s` or `.ts` segments for transmission. Therefore, the core is to capture the **MPD link**.

![](https://image.906732.xyz/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-01-15%20140946%20(1).png)

However, **DASH** is just a transmission protocol and is not responsible for encryption. So, we need to further investigate its encryption method. Open the console with F12 and find the "license" related request (https://mlic.dmm.co.jp/drm/clearkey/license):

![](https://image.906732.xyz/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-01-15%20142459.png)

**This is the standard format for ClearKey**: JWK (JSON Web Key). It's very straightforward, directly placing the Key ID (`kid`) and the Key (`k`) in an array:

- `kid` is the ID of the key, used to identify which stream this key corresponds to. Convert this base64 to hex: `GcbW7lIkNvGzbMd1M5GmfQ` -> `19c6d6ee522436f1b36cc7753391a67d`.

- The `k` string is very long (about 250+ characters). In standard ClearKey, the key is usually 16 bytes (32 Hex digits). Therefore, such a long `k` value often indicates that DMM has encrypted the real key in the response, or this is a wrapped token. However, for a large platform like DMM, specific decryption scripts or tools (such as dedicated browser extensions/scripts) are typically required to process this long string to obtain the original 32-digit Hex key. Therefore, **it's very difficult to try and crack the key manually; the best method is still "memory grabbing".**

ClearKey Mechanism: As the name implies, it's "clear key." Although the `k` value might be wrapped in a complex way during transmission (the extremely long string mentioned above), the W3C standard stipulates: **The browser must be able to convert this data back into a standard AES-128 key and pass it to the browser's Content Decryption Module (CDM).**

Therefore, the **breakthrough point** is: The decryption logic for ClearKey is typically completed within the browser's user-space process. For the decryption engine to work, the system must have a **fully decrypted, 16-byte original key** present in memory at some moment.

Install the browser extension **EME Logger**, refresh the website, and you can obtain the corresponding logs:

![](https://image.906732.xyz/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-01-15%20144332.png)

Save it as a local file, open it, and search for `MediaKeySession.update`.

![](https://image.906732.xyz/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-01-15%20144709%20(1).png)

These hexadecimal values like `0x7b`, `0x22`, etc., are actually the **ASCII codes** for characters in computers. I discovered through conversion that this data block is actually a **JSON string**.

- `0x7b` = `{`

- `0x22` = `"`

- `0x6b, 0x65, 0x79, 0x73` = `keys`

After converting the entire data block, you get the standard **JWK (JSON Web Key)** structure:

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

Perform Base64URL to Hex conversion again:

**KID**: `GcbW7lIkNvGzbMd1M5GmfQ` $\rightarrow$ `19c6d6ee522436f1b36cc7753391a67d`

**Key**: `t55KG1DxXXXXXXXXXXXXXX` $\rightarrow$ `b79e4a1b50f15d75d75d75d75d75d75d`

Then we can use **N_m3u8DL-RE** to download the file:

![](https://image.906732.xyz/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-01-15%20150212.png)

After the download is complete, the video and audio are automatically merged to get the final file:

![](https://image.906732.xyz/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-01-15%20150513.png)

Finally, I have integrated this entire workflow into a Chrome extension: [Project Link](https://github.com/ianho7/dmm-download-helper?tab=readme-ov-file). It contains detailed introductions and the [download link](https://github.com/ianho7/dmm-download-helper/releases). If it helps you, feel free to give it a free Star. The demo video is as follows: [![Video Title](https://image.906732.xyz/output.gif)](https://image.906732.xyz/output.webm)
