# MUDlark Lua Scripting Guide

MUDlark includes a built-in Lua scripting engine modelled closely after Mudlet. If you've written scripts for Mudlet, most of what you already know will work here. This guide covers everything from your first script to wiring triggers, timers, and aliases into Lua.

---

## Table of Contents

1. [Overview](#overview)
2. [Getting Started - Scripts](#getting-started--scripts)
3. [The Lua Console](#the-lua-console)
4. [Lua in Triggers](#lua-in-triggers)
5. [Lua in Timers](#lua-in-timers)
6. [Lua in Aliases](#lua-in-aliases)
7. [API Reference](#api-reference)
   - [Sending Commands](#sending-commands)
   - [Output](#output)
   - [Triggers](#trigger-control)
   - [Timers](#timer-control)
   - [Aliases](#alias-control)
   - [GMCP](#gmcp)
   - [Info Panel & Floating Windows](#info-panel--floating-windows)
8. [Built-in Variables](#built-in-variables)
   - [line and matches](#line-and-matches)
   - [vars - Persistent Variables](#vars--persistent-variables)
   - [msdp - MSDP Data](#msdp--msdp-data)
   - [gmcp - GMCP Data](#gmcp--gmcp-data)
9. [The GMCP Event Handler](#the-gmcp-event-handler)
10. [Color Codes in cecho](#color-codes-in-cecho)
11. [Sandbox and Limitations](#sandbox-and-limitations)
12. [Examples](#examples)

---

## Overview

MUDlark runs a sandboxed Lua 5.4 VM per world connection. The engine:

- Starts automatically when you connect to a world.
- Runs any scripts marked **Autorun on Connect** before the first trigger can fire.
- Shares one VM across all your scripts, triggers, timers, and aliases - so functions defined in a Script are available to your triggers.
- Tears down cleanly when you disconnect.

The API follows Mudlet naming conventions where possible, so community resources and existing Mudlet scripts are a good reference even if they don't transfer 100%.

The biggest difference between mudlet's scripting and MUDlark's is that you store variables like 'vars.test = value' instead of using something like 'local test = value'.

The second biggest difference is the lack of built in functions. While I've added some like selectstring, echo, etc, there is not support for geyser windows and other items like that. There are specific functions for printing to the floating windows in MUDlark however. 

If there is a function or feature in the Lua engine that you would like, please drop into the discord or email me and I'll do my best to provide it. There are some limitations to how Lua is sandboxed within the app as to be restricted from writing to files within the iOS ecosystem. 
---

## Getting Started - Scripts

Scripts are standalone Lua files tied to a world. Use them to define helper functions, set up initial state, or run any code that should run at connect time.

### Creating a Script

1. Connect to (or just select) a world.
2. Open the **world menu** (⋯ or gear icon) and tap **Lua Scripts**.
3. Tap **+** to create a new script.
4. Give it a name, paste or type your Lua source, and choose whether it should:
   - **Be enabled** - disabled scripts are skipped entirely.
   - **Run automatically on connect** - the script runs before triggers begin processing, so any functions or globals it defines are immediately available.
5. Tap **Save**.

### Running a Script Manually

From the script editor, tap **Run Now** while connected. Output from `echo()` and `print()` appears in the main scrollback as system lines. Errors appear in red.

### Script Execution Order

Autorun scripts execute in the order they appear in the list (drag to reorder). If script B calls a function defined in script A, make sure A appears above B in the list.

---

## The Lua Console

The Lua Console is an interactive REPL - type a Lua expression or statement, hit the send button, and see the result immediately. It shares the same VM as your scripts, so you can call functions you defined there, inspect `vars`, and test expressions before adding them to a trigger.

**Opening it:** world menu → **Lua Console**.

The console is available only while connected (the Lua engine is active). If you open it while disconnected you'll see a notice to connect first.

| Prompt | Meaning |
|--------|---------|
| `>` | Your input |
| `=` | Return value of an expression |
| `!` | Error |

### Example session

```
> vars.hp
= 95
> msdp.HEALTH
= 95
> send("health")
(command sent to MUD)
> 2 + 2
= 4
```

---

## Lua in Triggers

Any trigger action can be set to **Run Lua Script**. When the trigger fires, MUDlark runs the Lua code in the action field with the matched line and captures available as `line` and `matches`.

### Adding a Lua action to a trigger

1. Open **Triggers** from the world menu.
2. Create or edit a trigger.
3. In the Actions section, add an action and change the type to **Run Lua Script** (look for the `{}` curly-braces icon).
4. Type your Lua in the text area that appears.
5. Save the trigger.

### Available context variables

| Variable | Value |
|----------|-------|
| `line` | The full matched line from the MUD |
| `matches[1]` | The entire match (same as `line` for simple patterns) |
| `matches[2]` | First wildcard or regex capture group |
| `matches[3]` | Second wildcard or regex capture group |
| `matches[n]` | …and so on |

> **Note:** MUDlark uses 1-based indexing for `matches`, the same as Mudlet. `matches[1]` is always the full match.

### Example - announce health on a vitals line

Pattern: `Your health is * and your mana is *.` (simple pattern, two wildcards)

```lua
local hp = matches[2]
local mp = matches[3]
echo("HP: " .. hp .. "  MP: " .. mp)
```

### Example - enable a follow-up trigger when combat starts

Pattern: `You begin combat with *.` (regex: `You begin combat with (.+)\.`)

```lua
vars.target = matches[2]
enableTrigger("CombatRound")
echo("{GREEN}Target: " .. vars.target .. "{RESET}")
```

---

## Lua in Timers

Timer actions can also be set to **Run Lua Script**. The code runs on every tick. There is no `line` or `matches` context (timers aren't triggered by MUD output), but everything else - `vars`, `msdp`, `gmcp`, helper functions - is available.

### Example - send a keep-alive every 60 seconds

Create a timer with interval **60 seconds** and one **Run Lua Script** action:

```lua
sendRaw("\n")
```

### Creating timers from Lua

You can also create timers dynamically from scripts or trigger code using `createTimer()`:

```lua
-- Fire once after 5 seconds
createTimer("PostConnect", 5, [[
    send("look")
    disableTimer("PostConnect")
]], true)
```

See [`createTimer`](#timer-control) in the API reference for the full signature.

---

## Lua in Aliases

Aliases can optionally run a Lua script instead of sending their replacement text. Enable this by checking **Run Lua script instead of sending replacement** in the alias editor.

When a Lua alias fires:
- The replacement text is **not** sent to the server. Your Lua code is responsible for sending anything it needs to.
- `matches[1]` is the full input line; `matches[2]`, `matches[3]`, etc. are the wildcard captures (same convention as triggers).

### Example - a smart attack alias

Pattern: `att *`

```lua
local target = matches[2]
if target == "" then
    echo("Usage: att <target>")
else
    vars.target = target
    send("attack " .. target)
    echo("{YELLOW}Attacking " .. target .. "{RESET}")
end
```

---

## API Reference

### Sending Commands

```lua
send(text)
```
Sends `text` to the MUD as if you typed it. Alias processing runs normally.

```lua
sendRaw(text)
```
Sends `text` directly to the server with no newline appended and no alias processing. Useful for keep-alives or protocol-level bytes.

---

### Output

```lua
echo(text)
```
Prints plain text to the main scrollback as a system line (grey, no highlighting). Equivalent to Mudlet's `echo()`.

```lua
cecho(text)
```
Prints colored text to the main scrollback. Supports MUDlark color codes - see [Color Codes in cecho](#color-codes-in-cecho).

```lua
print(...)
```
Standard Lua `print`. MUDlark routes it through `echo()`, joining arguments with a tab character. Convenient for quick debugging.

```lua
wait(seconds)
```
Suspends the current script for `seconds` seconds (fractional values like `0.5` are fine), then resumes exactly where it left off. **Does not block the MUD** - the app remains fully responsive during the wait. Other triggers, timers, and even the same trigger firing again all continue to run independently.

> **Note:** `wait()` can only be used inside a trigger, timer, alias, or standalone script. Calling it in the Lua Console will produce an error.

```lua
-- Example: send a command, wait 2 seconds, send another
send("cast recall")
wait(2)
send("north")
```

```lua
appendToPanel(text)
```
Appends a line to the Info Panel. Supports color codes.

```lua
appendToFloatingWindow(index, text)
```
Appends a line to one of the four floating windows (0-indexed: `0`–`3`). Supports color codes.

---

### Trigger Control

```lua
enableTrigger(name)
disableTrigger(name)
deleteTrigger(name)
```

Enables, disables, or permanently removes the trigger with the given name. The name must match exactly what you typed in the trigger's **Name** field.

---

### Timer Control

```lua
enableTimer(name)
disableTimer(name)
deleteTimer(name)
```

Enables, disables, or removes a timer by name.

```lua
createTimer(name, intervalSeconds, code, repeating)
```

Creates a new timer at runtime.

| Argument | Type | Description |
|----------|------|-------------|
| `name` | string | Unique identifier for this timer |
| `intervalSeconds` | number | Seconds between ticks (minimum 0.05) |
| `code` | string | Lua source code to run on each tick (as a string, not a function reference) |
| `repeating` | boolean | `true` to repeat; `false` to fire once. Defaults to `true` |

> **Important:** `code` must be a **string** containing Lua source, not a function value. Use a long-bracket string `[[ ... ]]` to avoid quote escaping issues.

```lua
-- Repeat every 30 seconds
createTimer("HealCheck", 30, [[
    if msdp.HEALTH and tonumber(msdp.HEALTH) < 50 then
        send("quaff healing")
    end
]], true)
```

Calling `createTimer` with a name that already exists replaces the old timer.

---

### Alias Control

```lua
enableAlias(name)
disableAlias(name)
deleteAlias(name)
```

Enable, disable, or remove an alias by name.

---

### GMCP

```lua
sendGMCP(package, data)
```

Sends a GMCP message to the server. `data` is an optional JSON string.

```lua
sendGMCP("Core.Ping", nil)
sendGMCP("Char.Login", '{"name":"Elara","password":"hunter2"}')
```

---

### Timing

```lua
wait(seconds)
```
Suspends the script for `seconds` seconds (fractions like `0.25` and `0.5` are supported), then resumes from the same point. The main thread is never blocked - the MUD client remains fully responsive during the wait, and multiple scripts can be waiting simultaneously without interfering with each other.

```lua
-- Send three commands with a delay between each
send("drink potion")
wait(1)
echo("{GREEN}Potion consumed.{RESET}\n")
wait(0.5)
send("say Refreshing!")
```

> `wait()` is only valid inside a trigger/timer/alias/script. Calling it from the Lua Console will error.

---

### Line Highlighting (selectString / fg / bg)

These functions let you recolour specific text on the trigger line that fired the script - useful for highlighting important words without changing the rest of the line.

```lua
selectString([windowName,] text, occurrence)
```
Selects a substring of the current trigger line. `occurrence` is 1-based, so `1` selects the first match, `2` the second, etc. `windowName` is accepted but ignored (MUDlark has one main window). Returns the 0-based character position of the match, or `-1` if the text was not found.

```lua
deselect()
```
Clears the active selection so subsequent `fg()`/`bg()` calls are no-ops.

```lua
fg(color)
```
Sets the **foreground** (text) color of the current selection. `color` can be a name string or a bare ANSI index (0–255).

```lua
bg(color)
```
Sets the **background** color of the current selection. Same color argument as `fg()`.

**Supported color names:** `black`, `red`, `green`, `yellow`, `blue`, `magenta`, `cyan`, `white`, `bright_black` / `gray`, `bright_red` / `light_red`, `bright_green`, `bright_yellow`, `bright_blue`, `bright_magenta` / `pink`, `bright_cyan`, `bright_white`, `orange` (214), `brown` (130), `purple` (5). Any ANSI 256-color index (0–255) is also accepted.

```lua
-- Colour "big monster" red whenever it appears
if selectString("big monster", 1) > -1 then
    fg("red")
end

-- Highlight the second occurrence of "Bob" in yellow on cyan
if selectString("Bob", 2) > -1 then
    fg("bright_yellow")
    bg("cyan")
end
```

```lua
replace(text)
```
Replaces the current `selectString` selection with `text`. Color codes (`{RED}`, `{#RRGGBB}`, etc.) are supported inline. There is no separate `creplace()`. After the replacement the selection is cleared.

```lua
deleteLine()
```
Removes the current trigger line from scrollback entirely. Useful for suppressing spam lines or filtering sensitive output.

```lua
-- Replace a matched word in-line with a colored version
if selectString(matches[2], 1) > -1 then
    replace("{BRIGHT_YELLOW}" .. matches[2] .. "{RESET}")
end

-- Delete lines containing a spam pattern entirely
deleteLine()
```

> `selectString`, `fg`, `bg`, `replace`, and `deleteLine` only operate on the line that triggered the current script. They have no effect in timer scripts or the Lua Console (no trigger line exists in those contexts).

---

### Info Panel & Floating Windows

```lua
appendToPanel(text)
```
Appends to the right-side Info Panel. Color codes supported.

```lua
appendToFloatingWindow(index, text)
```
Appends to floating window `index` (0 = Window 1, 1 = Window 2, 2 = Window 3, 3 = Window 4). Color codes supported.

---

## Built-in Variables

### line and matches

These are set automatically before a trigger's Lua code runs.

```lua
line       -- The full text of the matched MUD line
matches[1] -- Full match (equals line for simple patterns)
matches[2] -- First wildcard or regex capture
matches[3] -- Second wildcard or regex capture
-- etc.
```

In timers and standalone scripts these are empty (`line = ""`, `matches = {}`).

---

### vars - Persistent Variables

`vars` is a key-value table that persists across sessions for the current world. Use it to remember state between connects, share data between triggers, or track cooldowns.

```lua
-- Set a value (saved automatically)
vars.lastRoom = "The Dark Forest"
vars.potionsUsed = tostring(3)

-- Read it back
echo(vars.lastRoom)
echo("Potions used: " .. (vars.potionsUsed or "0"))
```

> **Note:** `vars` stores only **string** values. Convert numbers with `tostring()` when writing and `tonumber()` when reading.

Values are debounce-saved (2 seconds after the last write) so rapid mutations in a trigger loop are batched into one disk write.

---

### msdp - MSDP Data

`msdp` is a read-only table populated automatically from MSDP variables sent by the server. Variable names match exactly what the server sends (usually all-caps).

```lua
echo("HP:  " .. (msdp.HEALTH or "?") .. "/" .. (msdp.MAX_HEALTH or "?"))
echo("MP:  " .. (msdp.MANA or "?") .. "/" .. (msdp.MAX_MANA or "?"))
echo("Room: " .. (msdp.ROOM_NAME or "unknown"))
```

If the server sends a complex value (array or object), `msdp.<var>` contains the raw JSON string.

---

### gmcp - GMCP Data

`gmcp` is a read-only table keyed by full GMCP package name. The value is the raw JSON string the server sent for that package.

```lua
local raw = gmcp["Char.Vitals"]   -- e.g. '{"hp":95,"mp":82}'
if raw then
    -- parse it yourself if you need individual fields
    echo(raw)
end
```

> **Heads up:** `gmcp[<package>]` gives you the raw JSON string, not a pre-parsed table. Use the standard `json` pattern below or a simple `string.match` for simple extractions.

---

## The GMCP Event Handler

Define a global function called `gmcpHandler` and it will be called automatically every time a GMCP message arrives, **in addition** to your triggers and GMCP Actions.

```lua
function gmcpHandler(package, json)
    if package == "Char.Vitals" then
        -- Quick-and-dirty extraction
        local hp = json:match('"hp"%s*:%s*(%d+)')
        if hp then
            vars.lastHP = hp
        end
    end
end
```

| Argument | Value |
|----------|-------|
| `package` | GMCP package name, e.g. `"Char.Vitals"` |
| `json` | Raw JSON string the server sent |

The handler is called inside a `pcall` so a Lua error in your handler won't break GMCP processing - it will appear as a red error line instead.

---

## Color Codes in cecho

`cecho()` and `appendToPanel()`/`appendToFloatingWindow()` support MUDlark color codes:

| Code | Color |
|------|-------|
| `{RED}` | Red |
| `{GREEN}` | Green |
| `{BLUE}` | Blue |
| `{YELLOW}` | Yellow |
| `{CYAN}` | Cyan |
| `{MAGENTA}` | Magenta |
| `{WHITE}` | White |
| `{BRIGHT_RED}` | Bright red |
| `{BRIGHT_GREEN}` | Bright green |
| `{BRIGHT_BLUE}` | Bright blue |
| `{BRIGHT_YELLOW}` | Bright yellow |
| `{BRIGHT_CYAN}` | Bright cyan |
| `{BRIGHT_WHITE}` | Bright white |
| `{#RRGGBB}` | Any hex color, e.g. `{#FF8800}` |
| `{RESET}` | Return to default color |

```lua
cecho("{GREEN}You healed for " .. amount .. " HP.{RESET}")
cecho("{#FF8800}[Combat]{RESET} " .. vars.target .. " hits you!")
```

---

## Sandbox and Limitations

MUDlark runs Lua in a restricted sandbox for security and stability.

### What's available

Standard safe libraries: `math`, `string`, `table`, `coroutine`, `utf8`, `tostring`, `tonumber`, `ipairs`, `pairs`, `pcall`, `xpcall`, `select`, `type`, `unpack`.

### What's blocked

The following are **not** available in scripts:

| Blocked | Why |
|---------|-----|
| `os.*` | File system and process access |
| `io.*` | File I/O |
| `debug.*` | VM internals |
| `load`, `loadstring`, `dofile`, `loadfile` | Dynamic code loading |
| `require` | Package loading |

Attempting to use any of these will result in a runtime error.

### Memory limit

Each Lua VM is capped at **50 MB** of memory by default. You can adjust this in **Settings → Lua Memory Limit**. Scripts that allocate beyond this limit will receive a runtime error rather than crashing the app.

### Infinite loops

There is no instruction-count safety net in the current build. A Lua script with an infinite loop (`while true do end`) **will hang the app**. If this happens, force-quit and reopen MUDlark - long press to edit your world and turn off "auto-login on fresh connect", then select your world/mud and remove the offending program.

Always use timers instead of `while` loops for repeated work:

```lua
-- Good: use a timer
createTimer("MyLoop", 1, [[ send("score") ]], true)

-- Bad: will hang the app
while true do
    send("score")
end
```

### Errors

Script errors appear as red lines in the main scrollback:

```
[Lua Error - Trigger:HealCheck] attempt to perform arithmetic on a nil value (global 'hp')
```

The format is `[Lua Error - <source>] <message>`. The source tells you which trigger, timer, or script caused the problem.

---

## Examples

### Track a cooldown with vars

```lua
-- In a trigger: "Your heal spell is ready."
function healReady()
    vars.healCooldown = "false"
    cecho("{GREEN}Heal is ready!{RESET}")
end
healReady()
```

```lua
-- In a trigger: "You cast a healing spell."
vars.healCooldown = "true"
createTimer("HealCooldown", 10, [[
    vars.healCooldown = "false"
    cecho("{GREEN}Heal cooldown expired.{RESET}")
    deleteTimer("HealCooldown")
]], false)
```

---

### Auto-loot after a kill

Pattern (simple): `* is dead.`

```lua
local mob = matches[2]
cecho("{YELLOW}" .. mob .. " is dead.{RESET}")
send("get all " .. mob)
send("look")
```

---

### Switch triggers between combat and travel modes

Put this in a Script (Autorun on Connect):

```lua
function combatMode()
    enableTrigger("AutoHeal")
    enableTrigger("CombatRound")
    disableTrigger("TravelPrompt")
    cecho("{RED}[Combat mode]{RESET}")
end

function travelMode()
    disableTrigger("AutoHeal")
    disableTrigger("CombatRound")
    enableTrigger("TravelPrompt")
    cecho("{GREEN}[Travel mode]{RESET}")
end
```

Then call `combatMode()` or `travelMode()` from any trigger or the console.

---

### GMCP: react to room changes

```lua
function gmcpHandler(package, json)
    if package == "Room.Info" then
        local roomName = json:match('"name"%s*:%s*"([^"]+)"')
        if roomName then
            vars.currentRoom = roomName
            appendToPanel("{CYAN}Room: " .. roomName .. "{RESET}")
        end
    end
end
```

---

### Repeating heal check timer

Put this in a Script (Autorun on Connect):

```lua
createTimer("AutoHeal", 3, [[
    local hp = tonumber(msdp.HEALTH) or 0
    local maxHp = tonumber(msdp.MAX_HEALTH) or 1
    if maxHp > 0 and (hp / maxHp) < 0.4 then
        send("quaff healing")
        cecho("{RED}[AutoHeal]{RESET} HP below 40%, healing!")
    end
]], true)
```

---

### Inspect any value in the console

```
> vars
= table: 0x...
> vars.currentRoom
= The Grand Market
> msdp.HEALTH
= 87
> gmcp["Char.Vitals"]
= {"hp":87,"mp":52,"sp":100}
```
