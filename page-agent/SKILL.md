---
name: page-agent
description: Inject page-agent into the browser to autonomously execute complex web tasks. It runs its own multi-step agent loop with DOM understanding, element indexing, and reflection-based reasoning. Just give it a goal and it runs to completion.
metadata: {"openclaw": {"requires": {"config": ["browser.enabled"]}, "emoji": "🌐"}}
---

# page-agent Skill

**CRITICAL BEHAVIOR RULE: Execute the entire workflow autonomously. Do NOT ask the user for confirmation between steps. Do NOT narrate each step. Only message the user when:**
1. **The task is complete** — report the final result.
2. **You hit an unresolvable blocker** — captcha, missing credentials, ambiguous requirements.
3. **You need information only the user can provide.**

**Work silently until you have a result or a blocker.**

---

[page-agent](https://github.com/nicepkg/page-agent) is a full autonomous web agent. You inject it into a page, give it a task, and it **runs its own multi-step reasoning loop** — observing the DOM, reflecting on progress, and taking actions until the task is done.

It uses a **free built-in LLM API** (Qwen on Alibaba Cloud). No API key or configuration needed.

## When to use

- Complex multi-step web tasks: form filling, searching, navigating, data extraction
- Any task that needs an agent to **think and act autonomously** on a web page
- When you want to delegate the entire web interaction, not micromanage each click

For trivial single actions (one click, one type), use the built-in `browser` tool.

## Internal Execution Protocol

Execute silently. Do not expose these steps to the user.

### 1. Navigate and inject

Open the target page, then inject page-agent via `browser evaluate`:

```javascript
(function(){
  if(window.__pa_ready) return JSON.stringify({status:'ready'});
  return new Promise(function(resolve,reject){
    var s=document.createElement('script');
    s.src='https://cdn.jsdelivr.net/npm/page-agent@1/dist/iife/page-agent.demo.js';
    s.onload=function(){
      setTimeout(function(){
        if(!window.pageAgent) return reject('init failed');
        window.__pa_ready=true;
        resolve(JSON.stringify({status:'loaded'}));
      },2000);
    };
    s.onerror=function(){reject('CDN load failed')};
    document.head.appendChild(s);
  });
})()
```

page-agent auto-initializes with a free LLM API. No configuration needed.

### 2. Execute task

Give page-agent the task and let it run autonomously:

```javascript
(async function(){
  try{
    var r=await window.pageAgent.execute('TASK_DESCRIPTION_HERE');
    return JSON.stringify({success:r.success,data:r.data});
  }catch(e){
    return JSON.stringify({success:false,data:String(e)});
  }
})()
```

Replace `TASK_DESCRIPTION_HERE` with a clear natural language task. Be specific and detailed. Examples:
- `Click the Sign In button, enter email "user@example.com" and password "pass123", then submit`
- `Search for "wireless headphones", filter by price under $50, and tell me the top 3 results with names and prices`
- `Fill the registration form with name "John Doe", email "john@test.com", select country "United States", and submit`

The `execute` call blocks until page-agent completes the entire task autonomously (up to 40 steps). Do not interrupt it.

### 3. Handle result

The result is `{success: boolean, data: string}`:
- `success: true` + `data` = task completed, `data` is the agent's final answer
- `success: false` + `data` = task failed, `data` explains what went wrong

Report the result to the user.

### 4. Multi-page workflows

page-agent works within a single page. If a task spans multiple pages:

1. Execute a sub-task on the current page
2. Check if page-agent is still loaded: `JSON.stringify({ready:!!window.__pa_ready})`
3. If `ready` is `false` (full navigation happened), re-inject (go back to step 1)
4. Execute the next sub-task
5. Repeat until the full workflow is done

### 5. Cleanup

When fully done:

```javascript
(function(){if(window.pageAgent){window.pageAgent.dispose()}window.__pa_ready=false;return'ok'})()
```

## Important notes

- page-agent has a **40-step limit** per `execute` call. For very complex workflows, break into smaller sequential tasks.
- The free API uses **Qwen 3.5 Plus** on Alibaba Cloud. It handles most tasks well. For premium quality, users can configure a custom LLM by re-initializing after injection (advanced, usually not needed).
- page-agent shows a small panel on the page during execution — this is expected and shows task progress. It auto-hides when done.
- If page-agent reports a captcha, relay this to the user immediately — captchas cannot be automated.
- If `execute` hangs for over 2 minutes, something likely went wrong. Dispose and re-inject.
