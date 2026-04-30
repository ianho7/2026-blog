---
title: MapToPoster Online：把你爱的城市搓成专业级艺术海报
published: 2026-04-30T06:52:46.732Z
description: ''
updated: ''
tags:
  - 创造与分享
draft: false
pin: 0
toc: true
lang: 'zh'
abbrlink: 'map-to-poster-online'
---
![ChatGPT Image 2026年4月29日 23_57_40.webp](https://imgf.0v0.one/landscape/webp/20260430_064848_2723.webp)
地图海报是一类很有意思的视觉作品：把一座城市的道路、水域、绿地等地理要素重新渲染成装饰画，而不是直接截一张普通地图。

我看到过一个 Python CLI 项目 [maptoposter](https://github.com/originalankur/maptoposter)，可以把城市地图生成海报，效果很有意思。但命令行工具对普通用户还是有一点门槛：要安装 Python、配置环境、运行命令，再去找输出文件。

所以我做了一个网页版：MapToPoster Online。它是 maptoposter 思路的浏览器端实现，目标是让用户打开网页后，就能选择城市、调整主题和字体，然后导出一张适合打印或做壁纸的城市地图海报。

- 在线体验：https://maptoposter.0v0.one
- GitHub：https://github.com/ianho7/maptoposter-online

## 它能做什么
MapToPoster Online 会从 OpenStreetMap 等数据源获取道路、水体、绿地等地理要素，再按海报风格重新渲染。
目前支持：
- 选择城市并生成地图海报
- 调整地图半径、主题、颜色、字体和版式
- 导出 A4 竖版、A4 横版、方形、手机壁纸、桌面等多种尺寸
- 300 DPI 输出，方便打印
- 20 种内置主题
- 上传自定义 TTF/OTF 字体
- 英文、中文、日文、韩文、德文、西班牙文、法文界面
- 浏览器 IndexedDB 缓存，重复生成同一城市会更快

![ops-coffee-1777452379237.webp](https://imgf.0v0.one/portrait/webp/20260430_070452_9509.webp)
![ops-coffee-1777452458845.webp](https://imgf.0v0.one/portrait/webp/20260430_070814_9963.webp)

## 真正的难点：浏览器里的大规模地图数据
这个项目的核心难点不是“画几条线”，而是在浏览器里处理足够大的地图数据。

以东京 18km 半径的测试数据为例，道路要素可以达到 56 万以上，原始 GeoJSON 大约 40MB。GeoJSON 可读性很好，但它本质上是大量嵌套对象。浏览器要解析、转换、传输这些对象时，很容易出现明显卡顿。

如果直接走传统路线，大致会是这样：
- 从接口拿到 GeoJSON
- JSON.parse
- 在 JavaScript 里转成对象数组
- 传给 Worker
- 再传给 WASM
- Rust 里再反序列化成结构体
- 最后绘制

每一步单独看都合理，但叠在一起就会很慢。项目早期优化记录里，JSON.parse 曾经会阻塞主线程 3 到 5 秒，UI 完全失去响应。JS 和 WASM 之间传复杂对象也会带来额外成本，因为会在两种运行时之间创建大量中间结构。

这也是项目使用 Rust/WASM 渲染的原因之一：把大规模几何计算和绘制放到更可控的内存模型里，并尽量减少 JavaScript 对复杂对象的处理。
![ChatGPT Image 2026年4月29日 21_30_24.webp](https://imgf.0v0.one/landscape/webp/20260430_064848_623.webp)

## 技术方案：尽早把地图压平成连续内存
项目现在的核心思路是：尽量少在 JavaScript 里创建复杂对象，尽早把地理数据压平成连续内存。

我把复杂的 GeoJSON 转成一个 Float64Array，大致结构类似这样：
```
[要素总数, 类型 1, 点数 N, x1, y1, ..., xN, yN, 类型 2, 点数 M, ...]
```
这样做的好处是，数据可以作为 Transferable Object 在线程之间传递，减少大对象复制；Rust/WASM 端也可以按偏移顺序读取，不需要重新还原成一堆嵌套结构体。

项目曾经评估过 MessagePack 这类二进制序列化方案。MessagePack 可以理解成一种更紧凑的 JSON 替代格式，它能减少文本解析开销，但如果后续仍然要反序列化成大量对象，瓶颈并不会彻底消失。

最终更有效的方向不是“把 JSON 换成另一种格式”，而是减少对象本身：让数据在 JS、Worker、WASM 之间尽可能保持为连续数组。
![ChatGPT Image 2026年4月29日 21_32_57.webp](https://imgf.0v0.one/landscape/webp/20260430_064848_8270.webp)

## 分片并行：按道路边界切开数据
地图投影本质上是大量坐标点的数学计算。每条道路之间基本独立，非常适合并行处理。

因此项目把大块二进制数据按道路边界切成多个 shard，交给多个 Worker 并行计算。

这里不能按固定字节数硬切。道路是一条 polyline，如果从中间切开，后续渲染就可能出现断裂。因此分片前需要先扫描索引，找到每条道路的边界，再决定每个 shard 的起止位置。

在东京 18km 测试数据中，早期 Worker 解析和处理耗时大约 6.5 秒。改成二进制加 4 核分片后，这部分降到了大约 0.95 到 1.05 秒。
![ChatGPT Image 2026年4月29日 21_35_35.webp](https://imgf.0v0.one/landscape/webp/20260430_064848_9676.webp)

## WASM 渲染器：一次扫描完成分类
道路渲染还有一个容易被忽略的点：不要为了不同道路类型反复扫描同一批数据。

直觉写法可能是：motorway 扫一遍，primary 扫一遍，secondary 再扫一遍，residential 再扫一遍。这样代码容易理解，但同一块内存会被重复遍历。

现在的做法是单次扫描：读取一条道路后，根据 road type 直接分发到对应的 PathBuilder。这样一次遍历就能完成分类，减少内存带宽占用，也更利于 CPU cache。

项目技术记录中，WASM 核心渲染从约 12 秒优化到 7.25 秒，这里面单次扫描是关键优化之一。
## 踩坑 1：300 DPI 不是把画布放大就结束了
做打印图时，很容易以为“分辨率乘上去”就可以了。比如 A4 300 DPI，会得到一个很大的画布。

但画布变大以后，原来的线宽也会显得不对。早期版本里，Residential、Tertiary 等小道路在高分辨率输出中几乎看不见。根本原因是线宽没有跟输出分辨率做补偿。

解决方向是把道路宽度和导出分辨率关联起来。不同道路等级仍然保持相对粗细，但整体需要根据输出 scale 调整。否则预览图看着还行，一到高分辨率导出，小街道就会消失。
## 踩坑 2：Overpass API 不是稳定的“大数据库接口”
地图数据主要来自 OpenStreetMap，查询道路、水体和绿地时会用到 Overpass API。

Overpass 很强，但它不是为“无限制在线生成大范围地图海报”准备的。范围一大，就可能遇到超时、节点繁忙、返回失败等情况。
项目里做了几件事来降低失败率和等待时间：

- 查询面积过大时自动切分成小块
- 并发请求多个 Overpass 镜像节点，取最快响应
- 已获取的数据写入 IndexedDB，下次生成同一城市时直接走缓存
- 大半径场景下降低部分道路精度，减少无意义的亚像素细节

这些处理不能让网络问题完全消失，但能让工具在真实浏览器环境里更可用。
## 踩坑 3：缓存应该放在浏览器端认真做
同一个城市、同一个半径的数据，用户很可能会反复调整主题、颜色和字体。如果每次都重新请求地图数据，体验会很差，也会给公共 API 增加压力。

所以项目使用 IndexedDB 缓存地图数据。处理后的城市数据会压缩保存到本地，再次生成时可以直接读取。这样用户调样式时，等待主要集中在渲染，而不是反复下载同一批 OSM 数据。
## 当前架构
简化后，整个流程大概是这样：
![ChatGPT Image 2026年4月29日 21_35_35.webp](https://imgf.0v0.one/landscape/webp/20260430_064848_1430.webp)

```
城市搜索
  -> 获取坐标
  -> 请求 OSM / Protomaps / Overpass 数据
  -> 清洗和裁切地理要素
  -> 转成 Float64Array
  -> Worker 分片投影
  -> Rust/WASM 渲染
  -> 浏览器预览和导出
```
技术栈：
- React 19 + TypeScript + Vite
- Tailwind CSS v4 + Radix UI
- OpenStreetMap / Overpass API / Protomaps
- Rust + WebAssembly + tiny-skia
- Web Worker
- IndexedDB
Paraglide JS 国际化
## 已知局限
目前项目还有两个比较明显的限制。

- 大城市、大半径生成时仍然可能比较慢，尤其受 Overpass 节点状态影响。虽然项目做了分块、并发和缓存，但数据源网络状态仍然是不可完全控制的因素。

- 不同城市的 OSM 数据完整度不同，有些地方水域、绿地或 POI 数据可能不完整，最终效果也会受影响。
## 接下来
下一步我比较想继续完善地图元素控制，例如 POI、道路等级、水域样式等。现在工具已经能生成可用的城市地图海报，但如果要让它适应更多城市和更多审美偏好，细粒度控制会很重要。

如果你对 GIS、地图渲染、WASM 性能优化或者在线设计工具感兴趣，欢迎试用，也欢迎提 issue。

- 在线体验：https://maptoposter.0v0.one
- GitHub：https://github.com/ianho7/maptoposter-online

