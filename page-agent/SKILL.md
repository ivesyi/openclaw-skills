---
name: page-agent
description: Inject page-agent's PageController into the browser for intelligent DOM extraction, element indexing, and reliable web interactions. Provides simplified page representation and indexed element actions (click, input, scroll, select) without requiring a separate LLM.
metadata: {"openclaw": {"requires": {"config": ["browser.enabled"]}, "emoji": "🌐"}}
---

# page-agent Skill

**CRITICAL BEHAVIOR RULE: Execute the entire workflow autonomously. Do NOT ask the user for confirmation between steps. Do NOT narrate each step to the user. Only message the user when:**
1. **The task is complete** — report the final result.
2. **You hit a blocker you cannot resolve** — captcha, login credentials needed, ambiguous instructions.
3. **You need information only the user can provide** — e.g. which option to pick when multiple valid choices exist.

**Work silently until you have a result or a blocker. The user gave you the task — go do it.**

---

Inject [page-agent](https://github.com/nicepkg/page-agent)'s PageController into the OpenClaw-managed browser. It gives you **structured DOM understanding** and **indexed element actions**. You handle the reasoning — PageController is your eyes and hands.

## When to use

Use when you need multi-step web interaction or when `browser snapshot` output is too noisy. For simple one-click actions, use the built-in `browser` tool.

## Internal Protocol

The following is your internal execution protocol. Execute it silently — do not expose these steps to the user.

### Inject (once per page load)

Via `browser evaluate`:

```javascript
(function(){
  if(window.__pa_ready) return JSON.stringify({status:'ready'});
  return new Promise(function(resolve,reject){
    var s=document.createElement('script');
    s.src='https://cdn.jsdelivr.net/npm/page-agent@1/dist/iife/page-agent.demo.js';
    s.onload=function(){
      setTimeout(function(){
        if(!window.pageAgent) return reject('init failed');
        window.pageAgent.panel.hide();
        window.pageAgent.panel.show=function(){};
        var el=document.getElementById('page-agent-runtime_agent-panel');
        if(el) el.style.display='none';
        window.__pa_ready=true;
        resolve(JSON.stringify({status:'loaded'}));
      },2000);
    };
    s.onerror=function(){reject('CDN load failed')};
    document.head.appendChild(s);
  });
})()
```

After full page navigation (not SPA), check `window.__pa_ready` and re-inject if `false`.

### Read page state

```javascript
(async function(){
  var s=await window.pageAgent.pageController.getBrowserState();
  return JSON.stringify(s);
})()
```

Returns `{url, title, header, content, footer}`. The `content` field is the simplified DOM:

```
[33]<div>User form</div>
	*[35]<button aria-label='Submit form'>Submit</button>
	[36]<input placeholder='Email'/>
```

- `[index]` — numeric index for interaction (only indexed elements are interactive)
- `*[index]` — NEW element since last read (appeared after your last action)
- Tab indentation — parent-child relationship
- `header` — page URL, viewport size, scroll position
- `footer` — `"... N pixels below ..."` means more content exists; `"[End of page]"` means no more

### Actions

All via `browser evaluate`, all return `{success, message}`:

**Click**: `window.pageAgent.pageController.clickElement(INDEX)`
**Input**: `window.pageAgent.pageController.inputText(INDEX,'TEXT')`
**Select**: `window.pageAgent.pageController.selectOption(INDEX,'OPTION_TEXT')`
**Scroll down**: `window.pageAgent.pageController.scroll({down:true,numPages:1})`
**Scroll up**: `window.pageAgent.pageController.scroll({down:false,numPages:1})`
**Scroll right**: `window.pageAgent.pageController.scrollHorizontally({right:true,pixels:300})`
**Run JS**: `window.pageAgent.pageController.executeJavascript('...')`

Wrap each call: `(async function(){var r=await <CALL>;return JSON.stringify(r)})()`

### Execution loop

```
inject → read → reason → act → read → reason → act → ... → done
```

Execute this loop continuously without user interaction. Re-read after every action to verify results. If a full page navigation occurs, re-inject before continuing.

### Cleanup

When fully done: `(function(){if(window.pageAgent){window.pageAgent.dispose()}window.__pa_ready=false;return'ok'})()`

## Automation rules

- Re-read after every action. Never assume success.
- Only use indexes from the latest read. Stale indexes cause failures.
- `*[` elements are responses to your action — check them.
- After text input, check for autocomplete/suggestions before proceeding.
- Scroll only when footer shows more content exists.
- Elements with `data-scrollable` can be scrolled independently via the `index` parameter.
- Captcha → stop and inform user. You cannot solve captchas.
- Do not repeat the same failing action more than 3 times. Try an alternative.
- Do not click `target="_blank"` links (new tab).
- After full navigation, re-inject (check `__pa_ready`).
