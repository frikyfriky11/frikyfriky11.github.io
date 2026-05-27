+++
date = 2026-05-27T20:10:00+02:00
title = "The Bath-Time Bug: When unattended-upgrades Pulls the .NET Runtime Out From Under You"
description = "A FileNotFoundException for an assembly that was very clearly right there. The culprit wasn't my code, my packages, or even me — it was a background updater swapping the .NET runtime while my dev server idled. Here's the investigation, step by step."
authors = ["Stefano Previato"]
tags = [".NET", "Linux", "WSL", "debugging", "apt", "ClosedXML"]
categories = ["Debugging"]
+++

## TL;DR

If a long-running .NET dev server suddenly throws something like:

```
System.IO.FileNotFoundException: Could not load file or assembly
'System.Drawing.Primitives, Version=8.0.0.0, ...'. The system cannot find the file specified.
```

…for an assembly you can plainly see exists on disk, the most likely culprit is this:

- Ubuntu's `unattended-upgrades` silently upgraded your .NET runtime (e.g. `8.0.26` → `8.0.27`) **while your server was running**.
- The old runtime folder it was bound to got deleted out from under it.
- .NET loads many framework assemblies **lazily**, so everything looked fine until the *first* time a code path needed that one specific assembly — and then it went looking in a folder that no longer existed.

**The fix:** restart your `dotnet run`. No code change. That's it. The rest of this post is *how I figured that out*, because the method is more useful than the fix.

---

## The error that made no sense

I'm building a personal finance app called Argon. One of its features imports bank statements from Excel files, using the excellent [ClosedXML](https://github.com/ClosedXML/ClosedXML) library. I'd been hacking on it all afternoon with the dev server running via `dotnet run`.

I uploaded a statement, and the API handed me a `500`:

```
System.IO.FileNotFoundException: Could not load file or assembly
'System.Drawing.Primitives, Version=8.0.0.0, ...'
   at ClosedXML.Excel.XLWorkbook..ctor(Stream stream)
   at ...SparkasseV1Parser.ParseAsync(Stream file)
```

My first reaction was the normal one: *"What did I break?"* My second reaction, after staring at it, was confusion — because I hadn't touched anything related to Excel parsing, packages, or `System.Drawing` in weeks. That code was old and stable.

Here's the thing that makes this kind of bug worth a blog post: **the file it couldn't find was sitting right there on my disk.** I checked. `System.Drawing.Primitives.dll`, version 8.0, present and correct in the .NET shared framework folder. And yet the app insisted it was missing.

That contradiction — *"missing file that obviously isn't missing"* — turned out to be the entire key to the case.

---

## The method: treat every clue as something your answer has to explain

When a bug doesn't make sense, the temptation is to start randomly changing things until it goes away. Resist that. Instead, collect facts and look for the one that contradicts your assumptions, because **the real cause is whatever resolves the contradiction.**

### Step 1: Read the stack trace from the bottom up

The bottom of a stack trace is where things actually went wrong; the top is just the unwinding. This one told me three things instantly:

- It's an **assembly loading** failure, not a logic bug.
- It's triggered by **ClosedXML**, a third-party library.
- The missing piece is a **framework** assembly (`System.Drawing.*`), not one of my own DLLs.

So this was almost certainly an environment problem, not a "my code is wrong" problem.

### Step 2: Form a hypothesis, then try to kill it

First guess, the classic one: *"a dependency isn't being copied to the output folder."* So I checked the project file, the `deps.json` (the manifest .NET uses to know what to load), and whether the DLL physically existed.

It was all there. Hypothesis dead. And that's *good* — a killed hypothesis shrinks the search space. I now knew the file existed but the app couldn't reach it.

### Step 3: Reproduce it in isolation

I wrote a ten-line throwaway program that did nothing but open an Excel workbook with ClosedXML. It **worked perfectly.**

That single result was enormous. It proved that my code and my packages were fine *in a freshly started process*. Which meant the bug lived not in the source, but in the **specific, already-running process** that had served my request.

### Step 4: Notice the lazy-loading trap

Why hadn't I seen this earlier in the day? Because .NET loads many assemblies **lazily** — only the first time a code path actually needs them. ClosedXML only touches `System.Drawing.Primitives` when it opens a workbook. I hadn't opened one all afternoon. The server could happily run for hours looking healthy, with a landmine sitting on a code path I simply hadn't stepped on yet.

So the question sharpened to: **what changed between when this process started and now, such that it's looking for the runtime in the wrong place?**

### Step 5: Ask "who is allowed to change this, and where do they keep the receipts?"

This is the mindset shift I most want to pass on. When you suspect *something on the system changed*, don't guess — find the log of the thing that's allowed to make that change. Almost everything that mutates your machine keeps a record:

| Question | Where to look |
|---|---|
| What changed on the OS, and when? | `/var/log/dpkg.log`, `/var/log/apt/history.log` |
| Did something auto-update in the background? | `/var/log/unattended-upgrades/unattended-upgrades.log` |
| What changed in the code? | `git log`, `git log -p -S "searchTerm"` |
| What's running right now, and for how long? | `ps -eo pid,etime,cmd`, `docker ps` |
| When was this file last touched? | `stat <file>`, `ls -la` |

My `dotnet` lived at `/usr/lib/dotnet`. On Debian/Ubuntu, anything under `/usr/lib` is almost always installed by the system package manager — `apt`, which sits on top of the lower-level `dpkg`. And **`dpkg` logs every install, upgrade, and removal, with timestamps**, in `/var/log/dpkg.log`. That's an audit trail of exactly what changed on the machine and when.

So I checked it. And there it was:

```
2026-04-26 14:00  install  dotnet-runtime-8.0  8.0.26
2026-05-27 17:41  upgrade  dotnet-runtime-8.0  8.0.26 → 8.0.27
```

The .NET 8 runtime had been upgraded from `8.0.26` to `8.0.27` at **17:41 that very afternoon**. The old `8.0.26` folder — the one my running server had bound itself to at startup — was gone, replaced by `8.0.27`.

The error happened at **19:46**, about two hours later. The timeline fit perfectly.

---

## "But I didn't run any updates!"

This was my exact reaction. I genuinely hadn't typed `apt upgrade`. So who did?

Ubuntu ships a background service called **`unattended-upgrades`** whose entire job is to silently install security updates so you don't have to remember to. It was enabled on my system, and the `apt` history log named the culprit directly:

```
Start-Date: 2026-05-27  17:41:28
Commandline: /usr/bin/unattended-upgrade        ← not me!
Upgrade: ... dotnet-runtime-8.0 (8.0.26 → 8.0.27) ...
```

That `Commandline` line is the confession. It wasn't me — it was the automatic updater. And because Microsoft ships .NET runtime patches through the *security* channel, .NET gets swept up by `unattended-upgrades` on a regular basis.

A small "gotcha" worth knowing if you use WSL like I do: WSL itself doesn't auto-update your Linux packages, but the **Ubuntu distribution running inside it** does, because `unattended-upgrades` is on by default — exactly as it would be on a normal Ubuntu box.

---

## The bath-time finale

Here's the human detail that made the whole thing click, and honestly the reason this bug existed at all.

That afternoon I'd left the dev server running and stepped away for a couple of hours — I had to give my little son his bath. (Priorities.) While I was elbow-deep in bubbles, `unattended-upgrades` woke up at 17:41 and swapped the runtime. My server kept running, blissfully unaware: every assembly it had *already* loaded was still sitting in memory, so nothing complained.

Then I came back, resumed testing, and uploaded a bank statement — the *first* time all day that code reached for `System.Drawing.Primitives`. .NET dutifully went to fetch it from the `8.0.26` folder it remembered… which no longer existed. Boom.

The maddening part: if I'd uploaded that statement *before* the bath, it would have loaded cleanly from `8.0.26` and I'd never have seen a thing. The two-hour idle window was precisely what gave the auto-updater time to pull the rug, on a code path I happened not to touch until afterward.

---

## The fix (and the habit)

Restart the dev server. A fresh process binds to the current runtime (`8.0.27`), finds the assembly exactly where it expects, and everything works. No code change, no package change, no `.csproj` surgery.

The lasting takeaway is a tiny habit:

> **After any .NET runtime upgrade, restart your long-running `dotnet` processes.**

And if you'd rather not be surprised by background updates at all, you can inspect or disable them:

- See what the auto-updater has been up to: `cat /var/log/unattended-upgrades/unattended-upgrades.log`
- Disable automatic upgrades: set `APT::Periodic::Unattended-Upgrade "0";` in `/etc/apt/apt.conf.d/20auto-upgrades` — but then *you* own the job of applying security patches.

Personally, I'm leaving it on. Security patches are good, and "restart the server after an update" is a cheap habit to build.

---

## The real lesson

The fix here is one sentence. The *method* is the part worth keeping:

1. **Read the stack trace bottom-up** to find where it really failed.
2. **Form a hypothesis and try to disprove it** — a dead hypothesis is progress.
3. **Reproduce in isolation** to separate "my code" from "my environment."
4. **Chase the contradiction.** "A missing file that's clearly present" was the whole case — it meant the process was looking at a path that no longer existed.
5. **Ask who's allowed to change the system, and read their log.** `dpkg`, `apt`, `git`, `ps` — almost everything keeps receipts. Investigation is mostly knowing which receipt to pull.

The bug wasn't in my code. It wasn't even something I did. But finding that out taught me more than a quick fix ever would have — and now I know exactly which log to open the next time a file goes "missing" while sitting right in front of me.
