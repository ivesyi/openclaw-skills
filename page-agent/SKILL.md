---
name: page-agent
description: Inject page-agent's PageController into the browser for intelligent DOM extraction, element indexing, and reliable web interactions. Provides simplified page representation and indexed element actions (click, input, scroll, select) without requiring a separate LLM.
metadata: {"openclaw": {"requires": {"config": ["browser.enabled"]}, "emoji": "🌐"}}
---

# page-agent Skill

Inject [page-agent](https://github.com/nicepkg/page-agent)'s PageController into the OpenClaw-managed browser to get **structured DOM understanding** and **indexed element interactions**.

You (the agent) handle the reasoning. PageController is your **eyes and hands** for the web.

## When to use

- Complex web tasks requiring multi-step interaction with a page
- Pages where `browser snapshot` output is too noisy or hard to reason about
- Tasks needing reliable element targeting by numeric index

Do NOT use for simple single-action browser operations — use the built-in `browser` tool directly.

## Injection

Run via `browser evaluate` on the target page. Only needs to run once per page load:

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
        window.__pa_ready=true;
        resolve(JSON.stringify({status:'loaded'}));
      },1500);
    };
    s.onerror=function(){reject('CDN load failed')};
    document.head.appendChild(s);
  });
})()
```

After full page navigation (not SPA), check and re-inject:

```javascript
JSON.stringify({ready:!!window.__pa_ready})
```

## Reading the page

Get the full structured page state via `browser evaluate`:

```javascript
(async function(){
  var s=await window.pageAgent.pageController.getBrowserState();
  return JSON.stringify(s);
})()
```

Returns:

```json
{
  "url": "https://example.com/page",
  "title": "Page Title",
  "header": "Current Page: [Page Title](url)\nPage info: 1920x1080px viewport...\n[Start of page]",
  "content": "[1]<a>Home</a>\n[2]<button>Sign In</button>\n[3]<input placeholder='Search...'/>",
  "footer": "... 500 pixels below (1.2 pages) - scroll to see more ..."
}
```

### How to read `content`

The `content` field is the simplified DOM — this is the key output:

```
[33]<div>User form</div>
	*[35]<button aria-label='Submit form'>Submit</button>
	[36]<input placeholder='Email'/>
```

- `[index]` — numeric index for interaction. Only elements with `[index]` are interactive.
- `<type>` — HTML element type (button, input, a, select, div, etc.)
- Text inside tags — element description or visible text
- `*[index]` — element is NEW since the last `getBrowserState()` call (appeared after your last action). Pay attention to these.
- Indentation (tab `\t`) — parent-child relationship. Indented element is a child of the element above it.
- Elements WITHOUT `[index]` are static text, not interactive.

### How to read `header` and `footer`

- `header` contains page URL, viewport dimensions, scroll position
- `footer` tells you if there is more content below/above:
  - `"... 500 pixels below ..."` → there is more content, you can scroll down
  - `"[End of page]"` → no more content below
  - `"[Start of page]"` → no more content above

## Actions

All actions via `browser evaluate`. Each returns `{success, message}`.

### Click

```javascript
(async function(){
  var r=await window.pageAgent.pageController.clickElement(INDEX);
  return JSON.stringify(r);
})()
```

Replace `INDEX` with the element's numeric index from the content output.

### Input text

```javascript
(async function(){
  var r=await window.pageAgent.pageController.inputText(INDEX,'TEXT_HERE');
  return JSON.stringify(r);
})()
```

Clears the field and types the new text. After input:
- Check if suggestions/autocomplete appeared (re-read the page)
- You may need to press Enter or click a submit button to complete

### Select dropdown option

```javascript
(async function(){
  var r=await window.pageAgent.pageController.selectOption(INDEX,'OPTION_TEXT');
  return JSON.stringify(r);
})()
```

Select by the visible option text, not by value.

### Scroll vertically

```javascript
(async function(){
  var r=await window.pageAgent.pageController.scroll({down:true,numPages:1});
  return JSON.stringify(r);
})()
```

Parameters:
- `down`: `true` = scroll down, `false` = scroll up
- `numPages`: how many viewport heights to scroll (0.5 = half page, 1 = full page)
- `index` (optional): scroll within a specific scrollable element instead of the page

### Scroll horizontally

```javascript
(async function(){
  var r=await window.pageAgent.pageController.scrollHorizontally({right:true,pixels:300});
  return JSON.stringify(r);
})()
```

Useful for wide tables. Parameters: `right` (boolean), `pixels` (number), `index` (optional).

### Execute JavaScript

```javascript
(async function(){
  var r=await window.pageAgent.pageController.executeJavascript('return document.title');
  return JSON.stringify(r);
})()
```

For advanced operations not covered by other actions.

## Workflow pattern

1. Navigate to page → Inject page-agent
2. **Read** the page (`getBrowserState`)
3. **Reason** about the content — decide what to do next
4. **Act** — click, input, scroll, or select
5. **Read** again — check the result of your action
6. Repeat 3-5 until the task is done

## Rules for reliable automation

- Always re-read the page after an action to verify it worked. Never assume success.
- If expected elements are missing after an action, try scrolling or check if the page changed.
- Only use indexes that are visible in the current content output. Re-read before acting on stale indexes.
- Elements marked with `*[` are new since your last read — these often indicate a response to your action (dropdowns opening, forms appearing, etc.).
- After inputting text, check for autocomplete suggestions. You may need to click a suggestion or press Enter.
- Scroll only when `footer` indicates more content exists below/above.
- Elements with `data-scrollable` attribute can be scrolled independently (use the `index` parameter in scroll).
- If a captcha appears, inform the user — you cannot solve captchas.
- Do not repeat the same action more than 3 times without progress.
- Do not click links with `target="_blank"` as they open new tabs.
- After full page navigation, re-inject page-agent (check `window.__pa_ready`).

## Cleanup

When done with all web automation:

```javascript
(function(){
  if(window.pageAgent){window.pageAgent.dispose()}
  window.__pa_ready=false;
  return 'cleaned';
})()
```
