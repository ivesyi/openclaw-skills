---
name: page-agent
description: Inject page-agent into the current browser tab for intelligent web page automation. Understands DOM structure, indexes interactive elements, and executes complex multi-step web tasks autonomously via an AI agent loop.
metadata: {"openclaw": {"requires": {"config": ["browser.enabled"]}, "primaryEnv": "PAGE_AGENT_API_KEY", "emoji": "🌐"}}
---

# page-agent Skill

Use [page-agent](https://github.com/nicepkg/page-agent) to perform intelligent, multi-step web automation tasks on any page in the OpenClaw-managed browser.

page-agent provides:
- **DOM simplification pipeline**: converts complex pages into concise text that an LLM can reason about
- **Indexed element system**: every interactive element gets a numeric index for reliable targeting
- **Autonomous agent loop**: reflection-before-action mental model with memory across steps
- **Tools**: click, input text, select dropdown, scroll (vertical/horizontal), execute JavaScript, wait, ask user

## When to use

- The user asks to **perform a complex, multi-step web task** (e.g. "fill out this form", "search for X and add the first result to cart")
- Standard `browser` snapshot + click/type is insufficient because the page is complex or multi-step reasoning is needed
- You need to delegate a web task and get a structured result back

Do NOT use for simple single-action browser operations — use the built-in `browser` tool directly.

## Setup

Configure in `~/.openclaw/openclaw.json` (optional — works without config using the free testing API):

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

## Complete Usage Protocol

Follow these steps exactly. All code blocks are for `browser evaluate`.

### Step 1: Navigate to the target page

```
browser open https://example.com
```

### Step 2: Inject page-agent

Run via `browser evaluate`:

```javascript
(function() {
  if (window.__pa && window.__pa.ready) return JSON.stringify({status:'already_loaded'});
  window.__pa = { ready: false, pending: null, resolve: null };
  return new Promise(function(resolve, reject) {
    var s = document.createElement('script');
    s.src = 'https://cdn.jsdelivr.net/npm/page-agent@1/dist/iife/page-agent.demo.js';
    s.onload = function() {
      setTimeout(function() {
        if (!window.pageAgent || !window.PageAgent) {
          reject('page-agent failed to initialize');
          return;
        }
        window.pageAgent.panel.hide();
        window.pageAgent.onAskUser = function(question) {
          return new Promise(function(res) {
            window.__pa.pending = question;
            window.__pa.resolve = res;
          });
        };
        window.__pa.ready = true;
        resolve(JSON.stringify({status:'loaded'}));
      }, 1500);
    };
    s.onerror = function() { reject('failed to load page-agent from CDN'); };
    document.head.appendChild(s);
  });
})()
```

### Step 3: (Optional) Reconfigure with custom LLM

Only if `PAGE_AGENT_*` env vars are set. Skip if using the free testing API.

```javascript
(function() {
  if (!window.PageAgent) return JSON.stringify({error:'not_loaded'});
  var oldAskUser = window.pageAgent.onAskUser;
  window.pageAgent.dispose();
  window.pageAgent = new window.PageAgent({
    model: 'MODEL_NAME_HERE',
    baseURL: 'BASE_URL_HERE',
    apiKey: 'API_KEY_HERE',
    language: 'en-US',
    enableMask: false
  });
  window.pageAgent.onAskUser = oldAskUser;
  window.__pa.ready = true;
  return JSON.stringify({status:'reconfigured'});
})()
```

Replace `MODEL_NAME_HERE`, `BASE_URL_HERE`, `API_KEY_HERE` with actual env var values.

### Step 4: Execute a task

```javascript
(async function() {
  if (!window.__pa || !window.__pa.ready) return JSON.stringify({error:'not_loaded'});
  try {
    var result = await window.pageAgent.execute('TASK_DESCRIPTION_HERE');
    return JSON.stringify({success: result.success, data: result.data});
  } catch(e) {
    return JSON.stringify({success: false, data: String(e)});
  }
})()
```

Replace `TASK_DESCRIPTION_HERE` with the natural language task. Examples:
- `Click the Sign In button`
- `Fill the search box with 'OpenClaw' and press Enter`
- `Find the pricing table and extract all plan names and prices`
- `Navigate to Settings, then change the language to English`

### Step 5: Handle ask_user (if page-agent asks a question)

After starting a task, if the execute call does not resolve quickly, check for pending questions:

```javascript
(function() {
  if (!window.__pa) return JSON.stringify({pending:null});
  return JSON.stringify({pending: window.__pa.pending});
})()
```

If `pending` is not null, the agent is waiting for an answer. Provide it:

```javascript
(function() {
  if (window.__pa && window.__pa.resolve) {
    window.__pa.resolve('YOUR_ANSWER_HERE');
    window.__pa.pending = null;
    window.__pa.resolve = null;
    return JSON.stringify({answered: true});
  }
  return JSON.stringify({answered: false});
})()
```

### Step 6: Handle page navigation (re-injection)

If the task caused a full page navigation (not SPA), page-agent is lost. Detect and re-inject:

```javascript
(function() {
  return JSON.stringify({loaded: !!(window.__pa && window.__pa.ready && window.pageAgent && !window.pageAgent.disposed)});
})()
```

If `loaded` is `false`, go back to **Step 2** to re-inject before running the next task.

**Important**: SPA navigations (pushState/replaceState) do NOT require re-injection. Only full page reloads do.

### Step 7: Cleanup

When completely done with all web automation:

```javascript
(function() {
  if (window.pageAgent) { window.pageAgent.dispose(); }
  window.__pa = null;
  return JSON.stringify({cleaned: true});
})()
```

## Multi-task workflow

For tasks that span multiple pages:

1. Execute task on current page (Step 4)
2. Check if page-agent is still loaded (Step 6)
3. If not loaded (full navigation happened), re-inject (Step 2) and optionally reconfigure (Step 3)
4. Execute next task (Step 4)
5. Repeat until done
6. Cleanup (Step 7)

## Tips

- page-agent works best on **fully loaded** pages. Wait 1-2 seconds after navigation before injecting.
- The free testing API uses Qwen models on Alibaba Cloud. For better quality on complex tasks, configure GPT-4o or Claude via env vars.
- page-agent has a **40-step limit** per task. For very long workflows, break them into smaller sequential tasks.
- If `execute` returns `{success: false}`, check the browser console logs (`browser console` or `browser evaluate` with `console.log`) for details.
- The panel UI is hidden by default in this skill. If you need visual debugging, remove the `panel.hide()` call in Step 2.
