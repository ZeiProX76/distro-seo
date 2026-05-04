# distro-seo

> End-of-audit SEO + Core Web Vitals fixer for AI coding agents.

A Claude Code / agent skill that turns a list of failing audit checks (Lighthouse, PageSpeed Insights, Search Console, GEO scans) into committed code fixes. Framework-aware (Next.js, Astro, SvelteKit, Remix, Vite, plain HTML).

## Install

```bash
npx skills add ZeiProX76/distro-seo
```

Or pick a specific scope:

```bash
npx skills add ZeiProX76/distro-seo@distro-seo
```

Works with Claude Code, Cursor, Codex, Cline, and any agent that loads `SKILL.md` files via the `skills` CLI.

## What it covers

**Performance / Core Web Vitals**
- LCP, INP, CLS, FCP, TTFB diagnostics + fixes
- Render-blocking resources, unused JS / CSS
- Image sizing, next-gen formats (AVIF, WebP), cache policy
- Long main-thread tasks, third-party code, font loading

**On-page SEO**
- Title tags, meta descriptions, H1 structure, alt text
- Internal links, content-to-code ratio, thin content
- Open Graph + Twitter Card

**Technical SEO**
- robots.txt, XML sitemap, canonical, HTTPS / mixed content
- Mobile viewport + tap targets, hreflang, structured data

**GEO (AI search visibility)**
- AI crawler access (GPTBot, PerplexityBot, ClaudeBot, Google-Extended, etc.)
- llms.txt
- Citability (claim front-loading, attribution, byline)
- JS-only rendering (the silent killer for AI crawlers)

34 issue recipes total. Each one is self-contained: diagnostic snippet, code fix, framework adapter, verification step.

## How to use

1. Run an audit (PageSpeed Insights, Lighthouse, our [DistributionMarket SEO scan](https://thedistributionmarket.com), or any third-party tool).
2. Hand the issue list to your agent: "Fix these: render-blocking resources, LCP > 4 s, missing meta description on /pricing, AI crawlers blocked".
3. The skill loads, matches each issue to a fix recipe, applies it surgically, and verifies.

## Why another SEO skill

Most existing SEO skills cover the *why*. This one is built for the *fix* phase. It assumes you already have an audit and want the diffs.

It is also one of the few that takes GEO (AI search) seriously: AI crawler matrix, llms.txt template, citability checks, JS-rendering detection.

## License

MIT. Built by [DistributionMarket](https://thedistributionmarket.com).
