---
title: "A Smart Bulb as a Claude Code Status Light"
date: 2026-08-04
categories:
- ai
- dev-tools
tags:
- claude-code
- hooks
- iot
- tuya
- smart-home
- ambient-computing
- shell-scripting

thumbnailImagePosition: left
thumbnailImage: /img/claude-code-lightbulb-hooks/thumbnail.png
---

Wiring a Wi-Fi smart bulb into Claude Code's hook system so it turns green while Claude works, red when it needs me, and white when it's idle - no terminal glance required.

<!--more-->

<p>
{{< youtube 9DO4e9aQuBo >}}
</p>

---

## The Problem

**[If you just want the code, scroll to Claude Code hook wiring.](#claude-code-hook-wiring)**

I run Claude Code across multiple terminal tabs and windows, often with long-running agentic turns - multi-file refactors, background subagents, tasks that chew through tool calls for a couple of minutes at a stretch. The failure mode is predictable: I tab away to do something else, forget which window is waiting on a permission prompt, and come back ten minutes later to find a session that's been sitting idle the whole time.

Claude Code already has a notification system - a bell, a badge, sometimes a terminal title change. None of that helps when the terminal itself isn't in view. What I wanted was something ambient - a signal I could catch from across the room without alt-tabbing through six windows to find out which one is stuck.

I had a Wipro NS9400 smart bulb sitting on a desk lamp, already on my Wi-Fi and already paired to the Tuya/SmartLife ecosystem. Claude Code hooks fire on session lifecycle events and can run arbitrary shell commands. Put those two facts together and the bulb becomes a status light for free.

## The Idea

Three colors, mapped to three states:

- **White** - session started, idle, or waiting for your next prompt
- **Green** - Claude is actively processing your prompt or running tools
- **Red** - needs immediate attention: a permission prompt or a notification is up

A small Python CLI (`light.py`) drives the bulb over the Tuya cloud API or directly over the LAN. A shell wrapper (`light-hook.sh`) sits between Claude Code's hook events and that CLI, translating each event into a color and filtering out noise from subagents.

```bash
python3 light.py -s on -b 70 -C green
python3 light.py --lan 192.168.1.100 -b 100 -C red
```

Code's here: [github.com/riyazwalikar/claude-lightbulb-hooks](https://github.com/riyazwalikar/claude-lightbulb-hooks)

## Claude Code hook wiring

Claude Code hooks are configured in `~/.claude/settings.json`. Each event runs a command and gets the event payload as JSON on stdin. Wiring the bulb in is just mapping events to colors.

Hooks can also be scoped to a single repo, in that repo's own `.claude/settings.json`. That's additive, not a replacement - Claude Code merges hook lists across scopes, so a project-level hook on the same event runs *alongside* the one in your user settings, not instead of it. The light wiring below lives in user settings because it's session-wide; project settings are for hooks specific to one repo.

```json
"hooks": {
  "SessionStart": [{ "hooks": [
    { "type": "command", "command": "\"~/.claude/hooks/light-hook.sh\" white" }
  ]}],
  "UserPromptSubmit": [{ "hooks": [
    { "type": "command", "command": "\"~/.claude/hooks/light-hook.sh\" green" }
  ]}],
  "PreToolUse": [{ "hooks": [
    { "type": "command", "command": "\"~/.claude/hooks/light-hook.sh\" green" }
  ]}],
  "PostToolUse": [{ "hooks": [
    { "type": "command", "command": "\"~/.claude/hooks/light-hook.sh\" green" }
  ]}],
  "Notification": [{ "hooks": [
    { "type": "command", "command": "\"~/.claude/hooks/light-hook.sh\" notify" }
  ]}],
  "PermissionRequest": [{ "hooks": [
    { "type": "command", "command": "\"~/.claude/hooks/light-hook.sh\" red" }
  ]}],
  "Stop": [{ "hooks": [
    { "type": "command", "command": "\"~/.claude/hooks/light-hook.sh\" white" }
  ]}],
  "SessionEnd": [{ "hooks": [
    { "type": "command", "command": "\"~/.claude/hooks/light-hook.sh\" white" }
  ]}]
}
```

`SessionEnd` fires on `/clear`, `/exit`, logout, or any other session termination - wiring it to white means the bulb doesn't sit green/red after you've walked away from the session entirely.

Two things in here needed more thought than the color-to-event mapping suggests.

### The subagent problem

Claude Code fires the same session lifecycle events for subagents as it does for your main session. Without a guard, a background subagent spawning mid-turn would fire its own `SessionStart` and repaint the bulb white while your main session is still very much working - and its `SessionEnd` would do the same thing on exit. The bulb would flicker between states that have nothing to do with what you're actually watching.

The fix is in `light-hook.sh`: check the hook payload for an `agent_id` field. If it's present, the event came from a subagent, and the hook exits without touching the light.

```bash
#!/usr/bin/env bash
set -euo pipefail
COLOR="$1"
INPUT="$(cat)"
AGENT_ID="$(echo "$INPUT" | jq -r '.agent_id // empty')"
if [ -n "$AGENT_ID" ]; then
  exit 0
fi

# "notify" decides from the payload: idle "waiting for your input"
# ping = white (session idle); anything else = red (act now).
if [ "$COLOR" = "notify" ]; then
  MESSAGE="$(echo "$INPUT" | jq -r '.message // empty')"
  case "$MESSAGE" in
    *"waiting for your input"*) COLOR="white" ;;
    *)                          COLOR="red" ;;
  esac
fi

cd /path/to/claude-lightbulb-hooks
exec python3 light.py -q --lan 10.10.10.100 -s on -C "$COLOR" -b 100
```

### The stale-red bug, and why `PostToolUse` fixes it

The `Notification` event fires for two very different things: a permission prompt going up, and the idle "waiting for your input" ping. Both look identical from the hook's side unless you read the message field, so `light-hook.sh notify` inspects the payload and picks white for the idle ping, red for everything else.

My first pass wired `PreToolUse` to green on the theory that "fires before every tool call" would reset the bulb the moment work resumed after a permission grant. It doesn't. `PreToolUse` fires once, *before* the permission check - not again after you approve it. The actual event order is `PreToolUse` (green) → `PermissionRequest` (red) → you click "Yes" → the tool just runs, with no hook in between to flip anything back. There is no `PermissionApproved` event in Claude Code's hook set - approval is invisible to hooks entirely. So the bulb sat red for the full duration of whatever you'd just approved, only clearing at `Stop` when the whole turn ended, which defeats the point: a red light telling you to act on a session that's already moved on.

The fix is `PostToolUse`, which fires right after a tool finishes executing - including the one you just approved. Wiring it to green means the bulb clears the instant that tool completes, not five tool calls or an entire turn later. It's not literally the moment you press Enter (there's no hook for that), but it's the closest available signal, and it's the difference between "stuck red until the agent stops" and "red for exactly as long as you're actually blocking it."

## Real example, with the noise included

Hooks for the same event just run as a list, in order - your light hook doesn't have to be the only thing listening. This is the actual `hooks` block from a working setup (paths swapped for placeholders), where a mode-tracking plugin and an `rtk` command wrapper run alongside the light call:

```json
"hooks": {
  "SessionStart": [
    {
      "hooks": [
        {
          "type": "command",
          "command": "\"<path-to-node>\" \"~/.claude/hooks/some-other-hook.js\"",
          "timeout": 5,
          "statusMessage": "Running some other hook..."
        },
        {
          "type": "command",
          "command": "\"~/.claude/hooks/light-hook.sh\" white",
          "timeout": 10
        }
      ]
    }
  ],
  "PreToolUse": [
    {
      "matcher": "Bash",
      "hooks": [
        { "type": "command", "command": "rtk hook claude" }
      ]
    },
    {
      "matcher": "",
      "hooks": [
        { "type": "command", "command": "\"~/.claude/hooks/light-hook.sh\" green", "timeout": 10 }
      ]
    }
  ],
  "PermissionRequest": [
    {
      "matcher": "",
      "hooks": [
        { "type": "command", "command": "\"~/.claude/hooks/light-hook.sh\" red", "timeout": 10 }
      ]
    }
  ],
  "PostToolUse": [
    {
      "matcher": "",
      "hooks": [
        { "type": "command", "command": "\"~/.claude/hooks/light-hook.sh\" green", "timeout": 10 }
      ]
    }
  ]
}
```

The `matcher` field lets you scope a hook to specific tools - `rtk` only fires on `Bash` calls, while the light hook uses an empty matcher so it fires on every tool. Nothing about adding the bulb required touching or reordering the hooks that were already there.

## Bulb setup

The bulb itself needs pairing through the SmartLife/Tuya app first, same as any smart home flow. After that, `light.py` talks to it two ways:

- **Cloud mode** - via Tuya's IoT platform. Works from anywhere with internet, no need to be on the same network as the bulb. Needs a Tuya Cloud project (`iot.tuya.com`) with the device linked, an access ID/secret, and a region.
- **LAN mode** - direct to the bulb's local IP. Faster, no round-trip through Tuya's servers, but you need the device's local key (also from the Cloud project page) and to be on the same network.

```json
{
    "device_id": "your-device-id",
    "api_key": "your-tuya-access-id",
    "api_secret": "your-tuya-access-secret",
    "api_region": "in",
    "device_local_key": "your-local-key",
    "device_version": 3.3
}
```

I run LAN mode - it's a status light, and I'd rather not add a round trip to Tuya's cloud on every tool call just to change a color. `light.json` holds the credentials and is gitignored; the repo ships a `light.json.example` template instead.

```bash
python3 light.py -s on -b 70 -C red
python3 light.py --lan 192.168.1.100 -b 60 -C yellow
python3 light.py -C '#ff6600'
```

### Getting `device_version` right

Guessing `device_version` was the single most annoying part of LAN setup. My Wipro bulb turned out to be `3.1`, not the `3.3` the config template defaults to, and a mismatch doesn't fail loudly - it decrypts fine and then rejects the framing.

`tinytuya`'s scan mode fixes this in one shot - it listens for the bulb's own broadcast and reports IP, device ID, local key, and protocol version together:

```bash
python3 -m tinytuya scan
```

```
Smart light   Product ID = key4fv3xs8twchhy  [Valid Broadcast]:
    Address = 10.10.10.109   Device ID = 63076103...  Local Key = Mm!*...  Version = 3.1
```

Whatever it reports as `Version` goes straight into `light.json` as `device_version`. It also writes a `snapshot.json` with the same data - gitignored, since it's got your local key in it.

## Not just a bulb

None of this is specific to a Wipro bulb, or even to light. A hook is just "run this shell command when this event fires" - the bulb is one particular endpoint I happened to have on my desk. Swap `light.py` for anything that can produce a signal from a shell command and the same event-to-color mapping becomes an event-to-*anything* mapping:

- A `PermissionRequest` hook that runs `afplay` or `paplay` on a short beep instead of (or alongside) turning the bulb red - useful if the light's out of your eyeline but you're still in earshot.
- A Bluetooth LE bulb or LED strip instead of Tuya/Wi-Fi - trade the LAN/cloud dance for `bluetoothctl`/`gatttool` or a vendor SDK, same three-color mapping.
- A `Stop` hook that fires a system notification, flashes a keyboard's RGB backlight, or flips a Philips Hue/LIFX via their local APIs - any of these are just another command in the same hook slot.

The lesson generalizes past the light: `PreToolUse` and `PostToolUse` fire around *every* tool call, `PermissionRequest` around every prompt, regardless of what's on the other end of the command. If you can script it, a Claude Code hook can turn it into a status cue.

## Caveats

This is a weekend hack, not a product:

- **It's specific to one bulb model and one hook shape.** The Tuya API surface (`tinytuya`) generalizes to most Tuya-based smart bulbs, but I've only tested the Wipro NS9400.
- **LAN mode is fragile to network topology.** Wi-Fi client isolation on some routers blocks LAN discovery entirely - cloud mode is the fallback when that happens.
- **Wrong `device_version` fails cryptically.** You'll see `Unexpected Payload from Device` (904) - the bulb decrypts the request fine but rejects the framing. Wrong local key instead gives `Check device key or version` (914). Both are solved by re-running `tinytuya scan` and re-pairing if the key doesn't match.
- **The subagent guard depends on `jq` being installed and on `PATH`.** If the light starts flickering to the wrong color mid-session, that's the first thing to check.
- **No retry logic.** If the bulb drops off Wi-Fi or Tuya's cloud API times out, the hook just fails silently rather than blocking your session - which is the right tradeoff for a status light, but worth knowing.

## Worth It?

It's a small thing. But an ambient signal that doesn't need a terminal glance changes how you use long agentic sessions - I now let turns run longer in the background because I trust the bulb to tell me when I actually need to look. Red across the room means look now. Anything else means keep doing what you were doing.

Full setup, wiring, and troubleshooting notes are in the repo: [github.com/riyazwalikar/claude-lightbulb-hooks](https://github.com/riyazwalikar/claude-lightbulb-hooks)

Until next time, Happy Hacking!
