# deslop.it

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Claude Code Plugin](https://img.shields.io/badge/Claude_Code-Plugin-blueviolet)](https://docs.anthropic.com/en/docs/claude-code)
[![Python](https://img.shields.io/badge/Python-3.x-yellow.svg)](https://python.org)
[![Website](https://img.shields.io/badge/Website-deslop.it-black)](https://deslop.it)

A Claude Code plugin that scans Python code for AI-generated bloat ("slop") across 6 categories: over-defensive code, premature abstractions, dead weight, verbose patterns, structural bloat, and documentation noise. Uses weighted scoring and offers auto-fix.

**Website:** [deslop.it](https://deslop.it)

## Install

```bash
claude plugin marketplace add zaffnet/deslop.it
claude plugin install deslop@deslop
```

## Usage

In any Python project, run:

```
/deslop <directory>
```

The skill scans all Python files, calculates a weighted slop density score, presents findings ranked by impact, and offers to auto-fix.

## Categories

| # | Category | Weight |
|---|----------|--------|
| 1 | Over-defensive code | 1.0x |
| 2 | Premature abstraction | 1.5x |
| 3 | Dead weight | 1.5x |
| 4 | Verbose patterns | 1.0x |
| 5 | Structural bloat | 1.5x |
| 6 | Documentation noise | 1.0x |

## Scoring

| Slop Density | Rating |
|---|---|
| 0–5% | Excellent — minimal slop |
| 5–15% | Good — minor cleanup possible |
| 15–30% | Needs work — significant bloat |
| 30%+ | Heavy slop — major cleanup needed |

## Development

### Website

The landing page is built with [Astro](https://astro.build) in the `site/` directory.

```bash
# Install dependencies
cd site && npm install

# Start dev server
npm run dev

# Build for production
npm run build

# Preview production build
npm run preview
```

### Deployment

The site deploys to Vercel. Configuration:

- **Root directory:** `site/`
- **Build command:** `npm run build`
- **Output directory:** `dist/`

### Project Structure

```
deslop.it/
├── .claude-plugin/        # Plugin manifest
│   ├── plugin.json
│   └── marketplace.json
├── skills/deslop/         # The deslop skill
│   ├── SKILL.md
│   └── references/
│       └── pattern-catalog.md
├── site/                  # Astro landing page
│   ├── src/
│   │   ├── layouts/Layout.astro
│   │   └── pages/index.astro
│   └── package.json
├── LICENSE
└── README.md
```

## License

MIT
