---
title: 'MapToPoster Online: Turn Your Favorite City into a Professional Art Poster'
published: 2026-04-30T06:52:46.732Z
description: ''
updated: ''
tags:
  - Create & Share
draft: false
pin: 0
toc: true
lang: 'en'
abbrlink: 'map-to-poster-online'
---
![ChatGPT Image 2026年4月29日 23_57_40.webp](https://imgf.0v0.one/landscape/webp/20260430_064848_2723.webp)
Map posters are a fascinating type of visual work: they take a city's roads, waterways, green spaces, and other geographic elements and re-render them as decorative art — rather than simply screenshotting an ordinary map.

I came across a Python CLI project called [maptoposter](https://github.com/originalankur/maptoposter) that generates city map posters with some really interesting results. But command-line tools still have a bit of a barrier for most users: you need to install Python, set up the environment, run commands, and then go find the output file.

So I built a web version: MapToPoster Online. It's a browser-based implementation of the maptoposter concept, designed so that users can open a webpage, choose a city, adjust the theme and fonts, and then export a city map poster suitable for printing or use as a wallpaper.

- Try it online: https://maptoposter.0v0.one
- GitHub: https://github.com/ianho7/maptoposter-online

## What It Does
MapToPoster Online fetches geographic elements — roads, bodies of water, green spaces, and more — from OpenStreetMap and other data sources, then re-renders them in a poster style.
Current features include:
- Select a city and generate a map poster
- Adjust map radius, theme, colors, fonts, and layout
- Export in multiple sizes: A4 portrait, A4 landscape, square, mobile wallpaper, desktop wallpaper, and more
- 300 DPI output for print-ready quality
- 20 built-in themes
- Upload custom TTF/OTF fonts
- UI available in English, Chinese, Japanese, Korean, German, Spanish, and French
- Browser IndexedDB caching — repeated generation of the same city is significantly faster

![ops-coffee-1777452379237.webp](https://imgf.0v0.one/portrait/webp/20260430_070452_9509.webp)
![ops-coffee-1777452458845.webp](https://imgf.0v0.one/portrait/webp/20260430_070814_9963.webp)

## The Real Challenge: Large-Scale Map Data in the Browser

The core challenge of this project isn't "drawing a few lines" — it's handling sufficiently large map datasets entirely within the browser.

Take Tokyo at an 18km radius as a benchmark: road features alone can exceed 560,000 elements, with the raw GeoJSON coming in at around 40MB. GeoJSON is human-readable, but it's fundamentally a mass of deeply nested objects. When the browser has to parse, transform, and transfer all of that, noticeable stuttering is almost inevitable.

If you went the traditional route, the pipeline would look roughly like this:
- Receive GeoJSON from the API
- JSON.parse
- Convert to an array of objects in JavaScript
- Pass to a Worker
- Pass to WASM
- Deserialize into structs in Rust
- Finally, render

Each step seems reasonable on its own, but stacked together they get slow fast. In early optimization logs for this project, `JSON.parse` alone was blocking the main thread for 3 to 5 seconds, leaving the UI completely unresponsive. Passing complex objects between JS and WASM also carries overhead, since it creates a large number of intermediate structures across two runtimes.

This is one of the reasons the project uses Rust/WASM for rendering: offloading large-scale geometric computation and drawing to a more controlled memory model, while minimizing JavaScript's exposure to complex object graphs.
![ChatGPT Image 2026年4月29日 21_30_24.webp](https://imgf.0v0.one/landscape/webp/20260430_064848_623.webp)

## The Approach: Flatten Map Data into Contiguous Memory as Early as Possible

The core idea now is to create as few complex JavaScript objects as possible and to flatten geographic data into contiguous memory as early in the pipeline as possible.

Complex GeoJSON gets converted into a `Float64Array` with a structure roughly like this:
```
[total_features, type_1, point_count_N, x1, y1, ..., xN, yN, type_2, point_count_M, ...]
```
The benefits: data can be transferred between threads as a Transferable Object, avoiding large object copies; and the Rust/WASM side can read sequentially by offset, without needing to reconstruct a pile of nested structs.

The project evaluated binary serialization options like MessagePack — essentially a more compact alternative to JSON that reduces text-parsing overhead. But if you still need to deserialize into large numbers of objects afterward, the bottleneck doesn't go away.

The more effective direction turned out not to be "swap JSON for another format," but to reduce objects altogether: keep data as contiguous arrays as it moves between JS, Worker, and WASM.
![ChatGPT Image 2026年4月29日 21_32_57.webp](https://imgf.0v0.one/landscape/webp/20260430_064848_8270.webp)

## Sharded Parallelism: Splitting Data at Road Boundaries

Map projection is fundamentally a math-heavy operation over a large number of coordinate points. Individual roads are independent of each other, which makes the problem highly parallelizable.

The project splits the large binary buffer into multiple shards at road boundaries and distributes them across parallel Workers.

You can't just cut at fixed byte offsets. A road is a polyline, and cutting through the middle of one would produce broken segments during rendering. So before sharding, the code scans the index to locate each road's boundaries, and uses those to determine where each shard begins and ends.

In the Tokyo 18km benchmark, early Worker parsing and processing took around 6.5 seconds. After switching to binary data with 4-core sharding, that dropped to roughly 0.95–1.05 seconds.
![ChatGPT Image 2026年4月29日 21_35_35.webp](https://imgf.0v0.one/landscape/webp/20260430_064848_9676.webp)

## The WASM Renderer: Single-Pass Classification

There's a subtle but impactful optimization in road rendering: don't scan the same dataset multiple times for different road types.

The naive approach might scan once for motorways, once for primary roads, once for secondary roads, once for residential roads — easy to reason about, but the same memory block gets traversed repeatedly.

The current approach does a single pass: as each road is read, it's dispatched directly to the corresponding `PathBuilder` based on road type. One traversal handles all classification, reducing memory bandwidth usage and improving CPU cache behavior.

In the project's optimization logs, WASM core rendering time came down from roughly 12 seconds to 7.25 seconds, with single-pass classification being one of the key contributors.

## Pitfall 1: 300 DPI Isn't Just Scaling the Canvas Up

When building print output, it's tempting to think "just multiply the resolution" and call it done. A4 at 300 DPI gives you a very large canvas.

But once the canvas is larger, the original stroke widths look wrong. In early versions, residential roads, tertiary streets, and similar minor roads were nearly invisible in high-resolution output. The root cause: stroke widths weren't compensated for output resolution.

The fix was to tie road widths to the export scale. Different road classifications still maintain their relative thickness, but the overall widths need to scale with the output. Otherwise the preview looks fine, and the moment you export at high resolution, the side streets vanish.

## Pitfall 2: The Overpass API Isn't a Reliable "Big Data Endpoint"

Map data primarily comes from OpenStreetMap, queried via the Overpass API for roads, water, and green spaces.

Overpass is powerful, but it wasn't designed for "unlimited large-radius map poster generation at scale." Once the query area gets large, you start hitting timeouts, busy nodes, and failed responses.

A few measures help reduce failure rates and wait times:
- Automatically split large query areas into smaller chunks
- Concurrently request multiple Overpass mirror nodes and use the fastest response
- Cache fetched data in IndexedDB so subsequent generation of the same city skips the network
- At large radii, reduce precision on some road types to cut down on sub-pixel detail that doesn't meaningfully affect the output

These don't eliminate network issues entirely, but they make the tool substantially more usable in real browser environments.

## Pitfall 3: Browser-Side Caching Deserves Serious Attention

For a given city and radius, users are very likely to iterate through themes, colors, and fonts repeatedly. Re-fetching map data on every render would make for a poor experience and put unnecessary load on public APIs.

The project uses IndexedDB to cache map data. Processed city data is compressed and saved locally; on subsequent generation, it's read directly from cache. This means when users are adjusting styles, the wait is dominated by rendering — not re-downloading the same OSM data over and over.

## Current Architecture

Simplified, the full pipeline looks like this:
![ChatGPT Image 2026年4月29日 21_35_35.webp](https://imgf.0v0.one/landscape/webp/20260430_064848_1430.webp)

```
City search
  -> Fetch coordinates
  -> Request OSM / Protomaps / Overpass data
  -> Clean and clip geographic features
  -> Convert to Float64Array
  -> Worker shards for parallel projection
  -> Rust/WASM rendering
  -> Browser preview and export
```
Tech stack:
- React 19 + TypeScript + Vite
- Tailwind CSS v4 + Radix UI
- OpenStreetMap / Overpass API / Protomaps
- Rust + WebAssembly + tiny-skia
- Web Worker
- IndexedDB
- Paraglide JS for i18n

## Known Limitations

There are currently two notable constraints.

- Generation for large cities or large radii can still be slow, particularly depending on Overpass node availability. Sharding, parallelism, and caching help, but the data source's network state remains an uncontrollable variable.

- OSM data completeness varies by city. In some areas, water, green space, or POI data may be sparse, which affects the final output quality.

## What's Next

The next area I want to focus on is finer control over map elements — things like POIs, road classification visibility, and water styling. The tool already generates usable city map posters, but if it's going to handle a wider range of cities and aesthetic preferences well, granular control will matter a lot.

If you're interested in GIS, map rendering, WASM performance optimization, or online design tooling, feel free to give it a try — and issues are very welcome.

- Try it online: https://maptoposter.0v0.one
- GitHub: https://github.com/ianho7/maptoposter-online