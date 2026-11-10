# AI Explained, Animated

Short animated explainers for AI engineering concepts. Each one runs in the browser, needs no install, and walks through one idea scene by scene. The same animations become YouTube videos (long form and Shorts).

**Watch them live:** https://technaom.github.io/ai-explained-animated/

By Manohar Papasani.

## Explainers

| # | Topic | The question it answers | Live page |
|---|---|---|---|
| 1 | RAG reranking | Your RAG retrieves the top 5 chunks, but the answer is in chunk #12, and raising top-K to 20 overflows the context window. How does the right chunk reach the LLM? | [Open](https://technaom.github.io/ai-explained-animated/explainers/rag-rerank/) |

More are on the way. Suggestions are welcome as GitHub issues.

## How to use them

- **To learn:** open a page and let it play, or pause it and step through with **Back** and **Next** (keyboard: Space and the arrow keys). Every step has a caption.
- **To teach or present:** use the scene buttons to jump to the part you are explaining.
- **Offline:** each explainer is a single HTML file. Download it and open it in any browser.

## Repo layout

```
explainers/<topic>/index.html   one self-contained animated explainer per folder
index.html                      the gallery page (GitHub Pages)
docs/design.md                  the course-to-video agent framework (design)
```

## Licence

- Code (HTML, CSS, JavaScript, scripts): [Apache-2.0](LICENSE).
- Explainer content (text, narration scripts, videos): [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). You can share and adapt it, including in courses, as long as you credit "Manohar Papasani, AI Explained, Animated" and link back here.
