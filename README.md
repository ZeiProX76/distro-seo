# distro-seo

> End-of-audit SEO + Core Web Vitals fixer for AI coding agents.

A Claude Code / agent skill that turns a list of failing audit checks (Lighthouse, PageSpeed Insights, Search Console, GEO scans) into committed code fixes. Framework-aware (Next.js, Astro, SvelteKit, Remix, Vite, plain HTML).

## Install

```bash
npx skills add ZeiProX76/distro-seo
```

Works with Claude Code, Cursor, Codex, Cline, and any agent that loads `SKILL.md` files via the [`skills`](https://skills.sh) CLI.

## What it covers

Four progressive-disclosure references, loaded only when needed:

| Reference | Domain | Coverage |
|---|---|---|
| `references/cwv.md` | Core Web Vitals + page speed | LCP, INP, CLS, FCP, TTFB, render-blocking, unused JS / CSS, image sizing, AVIF / WebP, cache policy, third-party code, payload, long tasks, font loading. **15 issues**. |
| `references/on-page.md` | On-page SEO | titles, meta descriptions, H1 hierarchy, alt text, internal links, content-to-code ratio, thin content, OG / Twitter Card. **8 issues**. |
| `references/technical.md` | Technical SEO | robots.txt, sitemap.xml, canonical, HTTPS / mixed content / redirects, mobile, hreflang, structured data. **7 issues**. |
| `references/geo.md` | Generative Engine Optimization | AI crawler matrix (GPTBot, PerplexityBot, ClaudeBot, Google-Extended, etc.), llms.txt, citability, JS-only rendering. **4 issues**. |

Each entry is a self-contained recipe: diagnostic snippet → code fix → framework adapter → verification step.

## How to use

1. Run an audit (PageSpeed Insights, Lighthouse, our [DistributionMarket SEO scan](https://thedistributionmarket.com), or any third-party tool).
2. Hand the issue list to your agent: *"Fix these: render-blocking resources, LCP > 4 s, missing meta description on /pricing, GPTBot blocked"*.
3. The skill loads, classifies each issue, loads only the matching reference, applies the fix surgically, and verifies.

## Why another SEO skill

Most existing SEO skills cover the *why* (here are the principles, here's the theory). This one is built for the *fix* phase. It assumes you already have an audit and want diffs.

It's also one of the few that takes **GEO (AI search visibility)** seriously: full crawler matrix, llms.txt template, citability checks, JS-rendering detection. As ChatGPT search, Perplexity, Claude, and Google AI Overviews send more referral traffic, this is the highest-leverage half of "SEO" in 2026.

## Structure

```
skills/distro-seo/
├── SKILL.md                  # router + workflow + targets
├── references/
│   ├── cwv.md                # CWV + perf
│   ├── on-page.md            # on-page SEO
│   ├── technical.md          # technical SEO
│   └── geo.md                # GEO / AI search
└── evals/
    └── evals.json            # 6 test cases for skill-creator-style eval loop
```

The skill follows the [skill-creator](https://github.com/anthropics/skills) progressive-disclosure pattern: a thin SKILL.md that routes to references on demand, so context isn't burned loading domains the active issue doesn't need.

## Evaluation

`skills/distro-seo/evals/evals.json` contains 6 test prompts spanning each domain plus a mixed multi-domain case and a judgment-call edge case. Run them with the skill-creator harness:

```bash
# from a checkout of skill-creator
python -m scripts.run_eval skills/distro-seo --eval-id 1
```

## License

MIT. Built by [DistributionMarket](https://thedistributionmarket.com).
