---
title: "PixelAssist Retrospective: ~2,100 Lines of TypeScript vs. ~40 Lines of Pixeltable"
url: "https://pixeltable.com/blog/pixelassist-retrospective-typescript-vs-pixeltable-backend"
date: "2026-04-16"
author: "team@pixeltable.com (Pierre Brunelle)"
feed_url: "https://pixeltable.com/blog/feed.xml"
---
We built an AI help assistant for pixeltable.com inside Next.js API routes. It works, but it grew to ~2,100 lines of TypeScript: tool loops, OPML parsing, URL allowlists, audit retries, a name-harvester, a link rewriter. Here is exactly what the same feature looks like written as a Pixeltable-native service (roughly 40 lines of core pipeline), with a side-by-side architecture diagram, a concern-by-concern comparison, an honest account of what Pixeltable cannot replace, and whether a future TypeScript SDK would change the answer.
