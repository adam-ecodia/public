# Claude Code - Comprehensive Tips & Tricks

A curated guide to getting the most out of Anthropic's Claude Code CLI tool.

---

## Table of Contents

- [Installation](#installation)
- [Configuration](#configuration)
- [Getting Started](#getting-started)
- [Core Workflow](#core-workflow)
- [Slash Commands](#slash-commands)
- [Effective Prompting](#effective-prompting)
- [Context & Memory](#context--memory)
- [Code Editing](#code-editing)
- [Testing](#testing)
- [Debugging](#debugging)
- [Project Understanding](#project-understanding)
- [Git Integration](#git-integration)
- [Best Practices](#best-practices)
- [Advanced Usage](#advanced-usage)
- [Common Mistakes to Avoid](#common-mistakes-to-avoid)
- [Avoiding Secret Exposure](#avoiding-secret-exposure)
- [Tips on Using Agents](#tips-on-using-agents)
- [Sources](#sources)

---

## Installation

```bash
# Via npm
npm install -g @anthropic-ai/claude-code

# Or via curl
curl -s https://claude.ai/install.sh | sh

# Verify installation
claude --version
```

---

## Configuration

### Environment Variables

```bash
export ANTHROPIC_API_KEY="sk-ant-..."       # Required for API access
export CLAUDE_CODE_MODEL="claude-sonnet-4-20250514"  # Optional: specify model
export CLAUDE_BASE_URL="https://api.anthropic.com"   # Optional: custom endpoint
export CLAUDE_CODE_DARK_MODE=true                    # Optional: dark theme
```

### Enterprise SSO Login Configuration

For enterprise environments using Single Sign-On (SSO):

**Option 1: OAuth/SSO Login**
```bash
# Initiate SSO login flow
claude auth login --sso

# Or via browser redirect
claude auth login --callback-url "https://your-org.auth0.com/login/callback"
```

**Option 2: Service Account (Machine-to-Machine)**
```bash
# For CI/CD and automated workflows
export ANTHROPIC_API_KEY="sk-ant-api-org-..."  # Organization API key
export CLAUDE_ORG_ID="your-org-id"              # Organization ID
```

**Option 3: SSO with SAML 2.0**
```bash
# Configure SAML in claude config (~/.claude/config.json)
{
  "auth": {
    "type": "saml",
    "saml": {
      "entryPoint": "https://your-org.okta.com/app/...",
      "issuer": "https://claude.ai",
      "callbackUrl": "https://claude.ai/auth/callback",
      "certificate": "/path/to/idp-cert.pem"
    }
  }
}
```

**Option 4: Okta/Azure AD/Google Workspace SSO**
```bash
# For common enterprise SSO providers
claude auth login --provider okta
claude auth login --provider azure
claude auth login --provider google-workspace

# Then configure in ~/.claude/config.json
{
  "auth": {
    "provider": "okta",
    "clientId": "your-client-id",
    "tenant": "your-org.okta.com"
  }
}
```

**Enterprise Proxy Configuration**
```bash
# If behind a corporate proxy
export HTTP_PROXY="http://proxy.company.com:8080"
export HTTPS_PROXY="http://proxy.company.com:8080"
export NO_PROXY="localhost,127.0.0.1,.company.com"

# Or configure in config.json
{
  "proxy": {
    "http": "http://proxy.company.com:8080",
    "https": "http://proxy.company.com:8080",
    "no_proxy": ["localhost", "127.0.0.1", ".company.com"]
  }
}
```

**Token Management for Enterprises**
```bash
# Refresh token automatically
claude auth refresh

# Check auth status
claude auth status

# Re-authenticate if token expired
claude auth login
```

**Security Best Practices for Enterprise**
- Store API keys in a secrets manager (AWS Secrets Manager, HashiCorp Vault)
- Use organization API keys for team deployments
- Set up IP allowlisting in Anthropic console
- Enable audit logging for team usage
- Rotate keys regularly

---

## Getting Started

- Run `claude-code` or `claude` in any project directory to launch
- Works with any language — not just Claude-specific
- First run prompts for API key setup (set `ANTHROPIC_API_KEY` env var)

---

## Core Workflow

- Ask in plain English — "fix the bug in auth.py"
- Be specific about scope — "refactor only the database layer"
- Use `/help` to see available commands

---

## Slash Commands

- `/browse` — Fetch and analyze web pages
- `/web` — Search the web
- `/think` — Toggle detailed thinking mode on/off
- `/claude` — Switch to Claude web interface
- `/review` — Start a code review
- `/test` — Generate and run tests
- `/search` — Search codebase
- `/lsp` — Query language server for code intelligence

---

## Effective Prompting

- Start with the file/function name: "In auth.py, add rate limiting"
- Specify the goal, not just the change: "Make this 50% faster" vs "add caching"
- Chain requests: "first do X, then update the tests, then verify it works"
- Use "as little as possible" to keep changes minimal
- Be conversational — it responds better to dialogue than single prompts
- Iterate — first pass is often rough, refine in follow-up messages

---

## Context & Memory

- `claude-code --context-file <file>` — Inject file contents into context
- Paste error messages directly — it reads stack traces
- Copy-paste code snippets for analysis
- `/clear` resets conversation but keeps project context

---

## Code Editing

- Say "diff" or "patch" to see changes before applying
- "Undo" reverts last change
- "Save" explicitly writes changes (some operations auto-save)
- Works with git — it can commit, branch, merge

---

## Testing

- "Write tests for..." generates test files
- "Run the tests" executes via shell
- Combine: "add the feature, write tests, run them, fix failures"

---

## Debugging

- Paste the full error/traceback — it analyzes context
- "Why is this slow?" profiles and explains
- "What's happening here?" walks through logic

---

## Project Understanding

- "What does this project do?" — reads README, structure
- "Map the architecture" — analyzes codebase relationships
- "Find where X is implemented" — searches and locates

---

## Git Integration

- "Commit the changes" — stages and commits
- "Create a PR for..." — drafts pull requests
- "Show me the diff" — reviews uncommitted changes
- "Revert the last 3 commits" — git operations

---

## Best Practices

1. **Be conversational** — it responds better to dialogue than single prompts
2. **Iterate** — first pass is often rough, refine in follow-up messages
3. **Verify** — always review generated code before accepting
4. **Scope carefully** — "fix my entire app" fails; "fix the login flow" works
5. **Use rollback** — if it goes wrong, say "undo" immediately
6. **Stay in terminal** — don't switch to browser mid-task
7. **Set context** — tell it the tech stack, coding style, constraints upfront
8. **Be specific** — vague requests get generic/broader changes than wanted
9. **Ask why** — it asks for clarification for a reason, listen to it
10. **Verify changes** — always review before accepting generated code

---

## Advanced Usage

- **Pipe output:** `claude-code --print "explain this" < file.py`
- **Diff mode:** `claude-code --diff` shows changes without applying
- **Non-interactive:** `claude-code --print "fix all lint errors"` for scripting
- **Attach multiple files:** `claude-code file1.py file2.py`
- **Context injection:** Feed custom context for specialized tasks
- **Shell pipes:** Chain with other CLI tools for powerful workflows

---

## Common Mistakes to Avoid

1. Asking too many things at once → breaks context
2. Not specifying language → may guess wrong
3. Vague requests → generic/broader changes than wanted
4. Ignoring its questions → it asks for clarification for a reason
5. Accepting changes blindly → always review before accepting
6. Switching tools mid-task → stay in terminal for best results

---

## Avoiding Secret Exposure

Claude Code has access to your codebase and can read/write files. Protect sensitive data:

### Before Sharing Code with Claude
- **Audit first:** Run `git log` and check for accidental commits
- **Remove .env files** from the working directory before starting
- **Use placeholders:** Replace real API keys with `YOUR_API_KEY_HERE`
- **Check git diff:** Review changes before staging — don't stage secrets

### Environment Variable Best Practices
```bash
# Store secrets in .env and add to .gitignore
# NEVER paste API keys directly in prompts

# Instead of:
# "Use this key: sk-ant-api03-xxx"

# Do this:
# "Use the ANTHROPIC_API_KEY env var I configured"
```

### Claude Code Security Features
- **Automatic secret detection:** Claude Code warns when it detects API keys in code
- **Reject suggestions:** Say "reject" if it suggests hardcoding secrets
- **Audit log:** Check `~/.claude/logs/` for any accidental exposures

### .gitignore Template for Claude Projects
```
# Environment files
.env
.env.local
.env.*.local
*.pem
*.key
credentials.json
token.json

# Secrets
secrets.yaml
secrets.yml
config-secrets.yml

# API keys
*.credentials
service-account.json
firebase-key.json
```

### What To Do If You Expose a Secret
1. **Immediately revoke** the exposed API key in the provider console
2. **Rotate keys** — generate new credentials
3. **Check git history** — `git log --all --source --remotes --oneline` to find when
4. **Use git filter-branch or BFG** to remove from history: `bfg --delete-files sensitive-file.txt`
5. **Force push** and notify team if organizational secrets were exposed

### Safe Patterns for Development
- Use `.env.example` with placeholder values (safe to commit)
- Use `python-dotenv` or similar to load from `.env` at runtime
- Inject secrets via environment at runtime, not at code time
- Use a secrets manager: AWS Secrets Manager, HashiCorp Vault, Doppler
- For local dev: use `direnv` to auto-load .env files when entering directories

---

## Tips on Using Agents

Claude Code excels as an autonomous coding agent. Here's how to maximize its effectiveness:

### Agent Mode Basics
- **Invoke agent mode** by adding `--agent` flag: `claude-code --agent`
- In agent mode, it autonomously plans and executes multi-step tasks
- It will ask for confirmation before major actions (configurable)

### Multi-Step Task Execution
- Give Claude a goal, not steps: "Set up a complete REST API with auth"
- It will: plan → write code → create tests → verify → iterate
- Chain subtasks: "First build X, then add Y, then deploy"
- Use checkpoints: "Do X and Y now, save Z for later"

### Tool Use in Agent Mode
- **READ** tool — read files before modifying
- **WRITE/EDIT** tool — create or modify files
- **BASH** tool — run shell commands, git, tests, builds
- **GREP** tool — search codebase for context
- **GLOB** tool — find files by pattern
- **WEB** tool — fetch docs, check APIs, research solutions

### Effective Agent Workflows
- **Bug fixes:** "Find and fix the memory leak in the auth module"
- **Feature development:** "Add real-time notifications with WebSockets"
- **Refactoring:** "Extract the database logic into a separate service"
- **Code review:** "Review the PR and suggest improvements"
- **Documentation:** "Write docs for the entire API surface"

### Best Practices for Agents
1. **Clear end goals** — tell it what success looks like upfront
2. **Provide context** — paste relevant code, errors, or files
3. **Set constraints** — "use Python 3.11", "follow this style guide"
4. **Review intermediate steps** — check its work as it goes
5. **Iterate** — "that approach is right, but try X instead"
6. **Let it explore** — give it room to figure things out
7. **Correct course** — "no, don't do it that way, use this approach"

### Agentic Patterns
- **Chain of thought:** Ask "think through this step by step" for complex tasks
- **Self-correction:** "You made an error in step 2, go back and fix it"
- **Verification loop:** "Run tests after each file change"
- **Parallel exploration:** "Find all files related to X while writing Y"

### Limitations & When to Intervene
- **Long tasks** — break into smaller chunks to avoid context overflow
- **Ambiguous requirements** — provide examples or reference implementations
- **Security-sensitive** — always review before running with elevated permissions
- **Large refactors** — do incrementally, not all at once

### Useful Flags for Agent Mode
- `--agent` — enable autonomous agent mode
- `--dangerously-skip-review` — skip confirmation prompts (use with caution)
- `--max-tokens` — limit response length for tight control
- `--resume <session>` — continue a previous agent session

---

## Sources

- https://docs.anthropic.com/en/docs/claude-code
- https://github.com/anthropics/claude-code
- https://claude.ai/code
