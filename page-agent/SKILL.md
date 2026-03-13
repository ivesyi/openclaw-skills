---
name: page-agent
description: Inject page-agent into the current browser tab for intelligent web page automation. Understands DOM structure, indexes interactive elements, and executes complex multi-step web tasks autonomously via an AI agent loop.
metadata: {"openclaw": {"requires": {"config": ["browser.enabled"]}, "primaryEnv": "PAGE_AGENT_API_KEY", "emoji": "🌐"}}
---

# page-agent Skill

Use [page-agent](https://github.com/nicepkg/page-agent) to perform intelligent, multi-step web automation tasks on any page in the OpenClaw-managed browser.

page-agent injects directly into a web page and provides:
- **DOM simplification pipeline**: converts complex pages into a concise text representation an LLM can reason about
- **Indexed element system**: every interactive element gets a numeric index for reliable targeting
- **Autonomous agent loop**: reflection-before-action mental model with memory across steps

## When to use this skill

- The user asks you to **perform a complex, multi-step web task** (e.g. "fill out this form", "find and click the pricing link", "search for X and add the first result to cart")
- Standard `browser` snapshot + click/type is insufficient because the page is complex or the task requires understanding page structure
- You need to delegate a web task and get a structured result back

Do NOT use this skill for simple single-action browser operations (one click, one type). Use the built-in `browser` tool directly for those.

## Setup

The skill needs an OpenAI-compatible LLM API to drive the page-agent reasoning loop.

Configure in `~/.openclaw/openclaw.json`:

```json5
{
  skills: {
    entries: {
      "page-agent": {
        enabled: true,
        env: {
          PAGE_AGENT_API_KEY: "your-api-key",
          PAGE_AGENT_BASE_URL: "https://api.openai.com/v1",
          PAGE_AGENT_MODEL: "gpt-4o"
        }
      }
    }
  }
}
```

If no env vars are set, page-agent falls back to its **free testing API** (rate-limited, evaluation only).

## Usage

### Step 1: Navigate to the target page

Use the `browser` tool to open the page you want to automate:

```
browser open https://example.com
```

### Step 2: Inject page-agent

Run this JavaScript via `browser evaluate` to load page-agent into the current tab:

```javascript
(function() {
  if (window.__pageAgentLoaded) return 'already loaded';
  return new Promise(function(resolve, reject) {
    var s = document.createElement('script');
    s.src = 'https://cdn.jsdelivr.net/npm/page-agent@1/dist/iife/page-agent.demo.js';
    s.onload = function() {
      window.__pageAgentLoaded = true;
      resolve('page-agent loaded');
    };
    s.onerror = function() { reject('failed to load page-agent'); };
    document.head.appendChild(s);
  });
})()
```

Wait ~2 seconds for initialization, then verify:

```javascript
!!window.pageAgent && !!window.PageAgent
```

### Step 3: (Optional) Reconfigure with custom LLM

If the user has set `PAGE_AGENT_*` env vars and wants to use their own LLM instead of the free testing API, reconfigure after injection:

```javascript
(function() {
  if (!window.PageAgent) return 'page-agent not loaded';
  if (window.pageAgent) window.pageAgent.dispose();
  window.pageAgent = new window.PageAgent({
    model: '__PAGE_AGENT_MODEL__',
    baseURL: '__PAGE_AGENT_BASE_URL__',
    apiKey: '__PAGE_AGENT_API_KEY__',
    language: 'en-US',
    enableMask: false
  });
  return 'reconfigured';
})()
```

Replace the `__PAGE_AGENT_*__` placeholders with the actual env var values.

If no custom config is needed, skip this step — the demo script auto-initializes with the free testing API.

### Step 4: Execute a task

```javascript
(async function() {
  var result = await window.pageAgent.execute('YOUR TASK DESCRIPTION HERE');
  return JSON.stringify({ success: result.success, data: result.data });
})()
```

The task description should be a natural language instruction, e.g.:
- `"Click the Sign In button"`
- `"Fill the search box with 'OpenClaw' and press Enter"`
- `"Find the pricing table and extract all plan names and prices"`
- `"Navigate to Settings, then change the language to English"`

### Step 5: Read the result

The execute call returns `{ success: boolean, data: string }`:
- `success: true` — task completed successfully, `data` contains the agent's final response
- `success: false` — task failed or was stopped, `data` contains the error or partial result

### Cleanup

When done, dispose the agent to free resources:

```javascript
if (window.pageAgent) { window.pageAgent.dispose(); window.__pageAgentLoaded = false; }
```

## Tips

- page-agent works best on pages that are fully loaded. Wait for the page to stabilize before executing tasks.
- For multi-page workflows, you may need to re-inject after navigation (check `window.__pageAgentLoaded`).
- The free testing API uses Qwen models hosted on Alibaba Cloud. For better quality, configure your own LLM.
- page-agent has a 40-step limit per task. For very long workflows, break them into smaller tasks.
- If a task fails, check the browser console for detailed logs.
