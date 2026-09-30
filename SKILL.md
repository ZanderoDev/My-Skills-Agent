---
name: thm-browser
description: "Drive TryHackMe in a real browser via browser-mcp: open /challenges and set filters (Red/Easy/Not Started/Free Only), join rooms, start/terminate lab machines, fill answer boxes. Use when the user asks to open TryHackMe, set challenge filters, join a room, start a machine, submit a flag, or automate any TryHackMe browser flow. Triggers: 'tryhackme', 'thm', 'set filter challenge', 'join room', 'start machine', 'browser-mcp tryhackme'."
---

## Primary Objective of the Task
When you are given a TryHackMe link, use the browser-mcp to open and view the challenge, title, and description, then search the internet for a writeup based on that challenge. **SEARCH WRITEUP ON THE INTERNET!!!**
Then solve it and enter the required flag on the challenge page using the browser-mcp.
When you run a command, if your technique is incorrect three times, you have to search the internet for a way to solve the problem.
YOU CAN ONLY STOP IF THE FLAG HAS BEEN INPUT INTO THE BROWSER.
Then, provide a brief final report summarizing the process from the beginning up to finding the flag.

## Me via browser-mcp

Field-tested procedures for controlling TryHackMe through `browser-mcp_*` tools.
Full field manual: `/root/CTF/Automation/Browser.md` (trik #1–#17).
Filter runbook source: `/root/CTF/Automation/thm-filter.md`.

## Stealth playbook (ALWAYS default when pentesting)

Low-noise, non-destructive enumeration. THM labs are isolated, but stay quiet anyway.

Baseline UA (reuse for all curl):
`Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0 Safari/537.36`

**Hard limits:** no DoS/flood/stress, no resource exhaustion, no destructive writes to
target files, no log wiping, no persistence. Enumerate → verify → report. Prefer one
targeted request over a broad sweep. Never probe a host other than the current target.

## Golden rules (never skip)

0. **NEVER manage the VPN.** The user owns VPN setup entirely: never install openvpn,
   never download/apply any `.ovpn`, never start/stop/restart any openvpn process, never
   edit VPN config or routing. If a lab target is unreachable from this terminal, REPORT
   the evidence (nmap output, route decision) and STOP — do not attempt any fix.
1. **NEVER touch AttackBox.** No `Start AttackBox`, no AttackBox terminal, no
   `attackbox` URLs. Pentest runs in the local terminal (bash), not in AttackBox.
3. `browser_list_tabs` first — reuse tabs, don't duplicate.
4. `browser_switch_tab` BEFORE any click (background tabs drop all mouse/keyboard input).
5. `browser_execute_script` takes parameter **`code`** as an EXPRESSION — wrap in `(() => { ... })()`. Never top-level `return`.
6. VERIFY every action via DOM data (`location.href`, `.value`, button labels). Never assume.
7. Can't view screenshots? Use DOM inspection instead of guessing.

## Workflow A — set /challenges filters (~11 calls, zero exploration)

Target: Type=Red, Difficulty=Easy, Status=Not Started, Free Only.
Menus are viewport-fixed (always open at y=326, option h=40) — coordinates below are frozen.

```
browser_navigate(url="https://tryhackme.com/challenges")   # note tab_id, ignore captcha if page loads
browser_switch_tab(tab_id=<above>)                          # MANDATORY before clicks
browser_click(selector="button:text(Type)")                 → browser_click_xy(x=72, y=426)    # Red
browser_click(selector="button:text(Difficulty)")           → browser_click_xy(x=287, y=426)   # Easy
browser_click(selector="button:text(Status)")               → browser_click_xy(x=387, y=466)   # Not Started
browser_click(selector="button:text(Subscription type)")    → browser_click_xy(x=553, y=466)   # Free Only
```

Menu maps (y = click row center): Type: All 346 / Purple 386 / Red 426 / Blue 466.
Difficulty: All 346 / (gap 386) / Easy 426 / Medium 466 / Hard 506.
Status: All 346 / Completed 386 / In Progress 426 / Not Started 466.
Subscription: All 346 / Premium Only 386 / MAX Only 426 / Free Only 466.

Verify (one `browser_execute_script`): filter bar must read
`["Red","Sort by","Easy","Not Started","Free Only"]`. Never click a filter button twice
(second click CLOSES the menu).

## Workflow B — join room + start machine

1. `browser_click(selector="button:text(Join Room)")` — verify it disappears.
2. `browser_click(selector="button:text(Start Lab Machine)")`.
3. If modal "You've reached your active machine limit": terminate each machine via its own
   `Terminate` button using `browser_click_xy` (text selector is ambiguous — buttons share
   text). RE-READ coordinates after each terminate (list re-renders, rows shift).
   Then `Close` → `Start Lab Machine` again.
4. Success signs: "Starting your machine… please wait!" + `Lab machine Status: On` +
   button becomes `Terminate`. IP appears after ~50–60s countdown — `sleep 55`, then regex
   `\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b` via script.

## Workflow C — fill + submit flag

1. Dump inputs first: `input, textarea, [contenteditable]` + name/id/placeholder.
   THM answer box is `<input name="1">` — selector `input#1` is INVALID CSS (leading-digit
   ID) and `browser_fill` fails SILENTLY (returns `null`).
2. `browser_fill(selector='input[name="1"]', value="THM{...}")` — success = `{ok:true,...}`.
3. ALWAYS read back `.value` via script before submit. `null` = silent fail, fix selector.
4. `browser_click(selector="button:text(Check)")`, read feedback text (`incorrect`/`correct`).

## Reading pages

- Full page text: `browser_get_page_content(format="text", max_chars=...)`.
- NEVER `document.body.innerText` for page reads — viewport-only, misses SPA scroll
  containers (proven: 1.738 vs ~2.300 chars on a THM room page).

## When tools misbehave

- "CDP did not respond / tab in background" → `switch_tab`, repeat ONE action.
- `landed:false` → check `location.href` first (navigation may have succeeded anyway).
- `Element not found` on SPA links → switch to `text=` selector (`a:text(Title)`).
- `ghost attach` repeats → `browser_reattach_debugger()`.
- Close tabs when done: `browser_close_tab(tab_id=...)`.
