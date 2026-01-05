# LLM Council Action

GitHub Action for multi-model consensus verification in CI/CD pipelines.

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-LLM%20Council-green?logo=github)](https://github.com/marketplace/actions/llm-council-quality-gate)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

## Overview

LLM Council provides AI-powered quality gates for your CI/CD pipelines. Instead of relying on a single model's judgment, it:

1. **Gathers opinions** from multiple LLMs (GPT, Claude, Gemini, etc.)
2. **Anonymizes and cross-evaluates** each response
3. **Synthesizes a consensus** verdict with confidence score

## Quick Start

```yaml
name: Quality Gate
on: [pull_request]

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: amiable-dev/llm-council-action@v1
        with:
          snapshot: ${{ github.sha }}
          confidence-threshold: 0.8
        env:
          OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `command` | Command: `gate`, `verify`, or `review` | No | `gate` |
| `snapshot` | Git SHA to verify | No | `${{ github.sha }}` |
| `file-paths` | Specific files to verify (space-separated) | No | - |
| `confidence-threshold` | Minimum confidence to pass (0.0-1.0) | No | `0.8` |
| `rubric-focus` | Focus: `Security`, `Performance`, `Testing`, `General` | No | `General` |
| `version` | llm-council-core version | No | `0.24.5` |
| `python-version` | Python version | No | `3.11` |
| `fail-on-unclear` | Fail when verdict is UNCLEAR | No | `false` |

## Outputs

| Output | Description |
|--------|-------------|
| `verdict` | `PASS`, `FAIL`, or `UNCLEAR` |
| `exit-code` | `0`=PASS, `1`=FAIL, `2`=UNCLEAR |
| `confidence` | Aggregate confidence score (0.0-1.0) |
| `summary` | Human-readable evaluation summary |
| `transcript-path` | Path to verification transcript |

## Exit Codes

| Code | Verdict | Action |
|------|---------|--------|
| `0` | PASS | Continue pipeline |
| `1` | FAIL | Block pipeline |
| `2` | UNCLEAR | Require human review (configurable) |

## Examples

### Security-Focused Code Review

```yaml
- uses: amiable-dev/llm-council-action@v1
  with:
    command: review
    file-paths: src/auth.py src/api.py
    rubric-focus: Security
    confidence-threshold: 0.85
  env:
    OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
```

### Verify Specific Files

```yaml
- uses: amiable-dev/llm-council-action@v1
  with:
    command: verify
    file-paths: src/feature.py tests/test_feature.py
    confidence-threshold: 0.75
  env:
    OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
```

### Strict Mode (Fail on UNCLEAR)

```yaml
- uses: amiable-dev/llm-council-action@v1
  with:
    fail-on-unclear: true
    confidence-threshold: 0.9
  env:
    OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
```

### Using Outputs

```yaml
- uses: amiable-dev/llm-council-action@v1
  id: council
  with:
    snapshot: ${{ github.sha }}
  env:
    OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}

- name: Comment on PR
  if: github.event_name == 'pull_request'
  uses: actions/github-script@v7
  with:
    script: |
      const verdict = '${{ steps.council.outputs.verdict }}';
      const confidence = '${{ steps.council.outputs.confidence }}';
      github.rest.issues.createComment({
        owner: context.repo.owner,
        repo: context.repo.repo,
        issue_number: context.issue.number,
        body: `## LLM Council: ${verdict}\nConfidence: ${confidence}`
      });
```

## API Keys

The action supports multiple LLM providers via environment variables:

| Variable | Provider |
|----------|----------|
| `OPENROUTER_API_KEY` | OpenRouter (recommended - access to 100+ models) |
| `OPENAI_API_KEY` | OpenAI directly |
| `ANTHROPIC_API_KEY` | Anthropic directly |

**Recommended**: Use OpenRouter for access to multiple models with a single API key.

## Setup Guide

### Step 1: Get an API Key

1. Sign up at [OpenRouter](https://openrouter.ai/) (recommended) or your preferred provider
2. Create an API key from the dashboard
3. Note the key for the next step

### Step 2: Add Secret to Your Repository

**Via GitHub UI:**

1. Go to your repository → **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Name: `OPENROUTER_API_KEY`
4. Value: Your API key from Step 1
5. Click **Add secret**

**Via GitHub CLI:**

```bash
gh secret set OPENROUTER_API_KEY --repo your-org/your-repo
# Paste your API key when prompted
```

### Step 3: Add Workflow

Create `.github/workflows/council-gate.yml`:

```yaml
name: Council Quality Gate
on: [pull_request]

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: amiable-dev/llm-council-action@v1
        with:
          snapshot: ${{ github.sha }}
          confidence-threshold: 0.8
        env:
          OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
```

### For Maintainers: Version Sync Setup

If you're maintaining a fork or want automatic version sync between `llm-council` and this action:

1. **Create a Personal Access Token (PAT)**
   - Go to https://github.com/settings/tokens
   - Click "Generate new token (classic)"
   - Name: `llm-council-action-sync`
   - Scopes: `repo` (full control)
   - Expiration: 90 days (or your preference)

2. **Add PAT to llm-council repo**
   ```bash
   gh secret set ACTION_REPO_PAT --repo your-org/llm-council
   # Paste the PAT when prompted
   ```

3. **How it works**
   - When a new version is released to PyPI, `sync-action.yml` triggers
   - It updates the default version in `action.yml`
   - It moves the `v1` tag to the latest commit

**Manual sync** (if not using ACTION_REPO_PAT):

```bash
# In llm-council-action repo
VERSION="0.24.6"  # New version
sed -i "s/default: '[0-9.]*'/default: '$VERSION'/" action.yml
git commit -am "chore: sync with llm-council-core v$VERSION"
git push origin main
git tag -f v1 && git push -f origin v1
```

## Step Summary

The action automatically generates a GitHub Step Summary with:

- Verdict and confidence score
- Threshold comparison
- Full evaluation output (collapsible)

## Performance

- **First run**: ~15-20s (pip install + cache warm-up)
- **Cached runs**: ~3-5s (cache hit)
- **Evaluation time**: Depends on council size (typically 30-60s)

## Related

- [llm-council-core](https://github.com/amiable-dev/llm-council) - Core library
- [Documentation](https://llm-council.dev) - Full documentation
- [ADR-034](https://github.com/amiable-dev/llm-council/blob/master/docs/adr/ADR-034-agent-skills-verification.md) - Agent Skills specification

## License

MIT - See [LICENSE](LICENSE)
