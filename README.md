# skills

Agent skills by Ritvik Kudalkar, for Claude Code and other agents that support the open Agent Skills format.

## Skills

| Skill | What it does |
|---|---|
| [tool-site-seo-check](skills/tool-site-seo-check/SKILL.md) | Audits small tool and calculator websites (Astro or similar static sites) for SEO and AdSense readiness: canonical tags, sitemap, duplicate pages, thin content, schema, internal links and legal pages. Includes a no-dependency audit script for the built site. |

## Install

```bash
npx skills add RitvikKudalkar/skills --skill tool-site-seo-check
```

Or copy `skills/tool-site-seo-check/` into `~/.claude/skills/` to make it available in every project.

## Use

In Claude Code, open your site's repo and ask:

> run tool-site-seo-check on this repo

The skill builds the site, runs the audit script, checks what the script can't, and reports issues as critical, important or nice-to-have, with the exact fix for each. It asks before changing anything.
