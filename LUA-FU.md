# Netdev 0x1A: the lua-fu behind the dropreason demo

This is the prompt log behind the demo shown at the
[*Your Fu is Better Than Mine, 3.0!*](https://netdevconf.info/0x1A/sessions/bof/your-fu-is-better-than-mine-30-bof.html)
BoF at Netdev 0x1A (Rome, July 2026), a session where networking folks trade the
tricks they actually use, with bonus points this year for AI skills and
effective prompts. The demo was vibe-coded, so our fu is this log: every
prompt below, translated from Portuguese and grouped by theme, steered an AI
assistant from the first request for ideas to the finished example.

The tool itself answers "why is my packet dying?". The kernel already knows:
dropped `sk_buff`s go through `kfree_skb_reason()`, and the reason is right
there in its arguments. The demo is a 57-line Lua script, comments included, that puts
a kprobe on that one function, reads the reason out of the probed arguments, and
names it with a table generated from the kernel's own headers.

Loaded, it counts drops system-wide by name, into a table queried live
from [lunatik](https://github.com/luainkernel/lunatik)'s in-kernel REPL. To
catch one reason in the act, uncomment a line and reload: the first hit prints
the registers and call trace of the exact drop site, the caller that dropped it,
and the process it happened under.

The code lives on lunatik's
[`netdev0x1A-fu`](https://github.com/luainkernel/lunatik/tree/netdev0x1A-fu)
branch: the example
([`examples/dropreason.lua`](https://github.com/luainkernel/lunatik/blob/netdev0x1A-fu/examples/dropreason.lua)),
the probe patch that exposes the probed function's arguments, and the autogen
spec that names the drop reasons;
[the whole diff](https://github.com/luainkernel/lunatik/compare/e53fb15e...netdev0x1A-fu)
is three commits.
The pre-squash history is on
[`netdev0x1A-fu-dev`](https://github.com/luainkernel/lunatik/tree/netdev0x1A-fu-dev),
with the review fixups still visible, the use-after-return fix from the audit
below among them.

---

## Reproducing it

The flow below was validated on kernel 6.8 (arm64). Build the branch and
install (`make install` installs the example):

```
git checkout netdev0x1A-fu
make && sudo make install && sudo lunatik reload
```

Arm the probe (`hardirq`: the kprobe fires in interrupt context) and open the
REPL:

```
sudo lunatik run examples/dropreason hardirq
sudo lunatik
> drops = require("examples.dropreason_report")
```

Cause a drop from another shell (UDP to a closed port) and read it back:

```
echo x > /dev/udp/127.0.0.1/1
```

```
> drops.NO_SOCKET
1
> return drops.report()
      1  NO_SOCKET
     55  NOT_SPECIFIED
```

To catch `NO_SOCKET` in the act, uncomment the `WATCH` line in the installed
copy (`/lib/modules/lua/examples/dropreason.lua`), reload the script, and
trigger again:

```
sudo lunatik stop examples/dropreason
sudo lunatik run examples/dropreason hardirq
echo x > /dev/udp/127.0.0.1/1
```

`dmesg` then has the drop caught in the act:

```
dropreason: caught NO_SOCKET, registers:
...
pc : kfree_skb_reason+0x0/0x1d8
lr : __udp4_lib_rcv+0x450/0x748
...
Comm: bash
```

The counters reset when the script reloads (fresh table); the `drops` binding in
the REPL keeps working, it reads the shared environment on every access.

## Finding the demo

- "I want to present something at this session
  <https://netdevconf.info/0x1A/sessions/bof/your-fu-is-better-than-mine-30-bof.html>
  to show how lunatik can be useful. I'd like suggestions, to put together an
  example. One case I considered was
  [*A Small Anecdote on Scriptable Operating Systems in Practice*](https://medium.com/@lourival.neto/a-small-anecdote-on-scriptable-operating-systems-in-practice-3a76fed83f97),
  but you don't have to limit yourself to it. I want alternatives. Think of
  something to impress the audience."
- "Put together the run sheet with your recommendation, whatever you find most
  interesting and impressive."

The assistant pitched a four-act run sheet:

- a 44-line HMAC chardev driver in Lua, reproducing live the case from the
  opening prompt
- a netfilter ping filter edited live into an alternating drop (exactly 50% ping
  loss)
- a filter written by AI on stage
- an rtnetlink example listing links and installing a route from kernel Lua

The review:

- "I think we should present only one of the acts. We don't want to steal all the
  attention. Which one is the most relevant?"
- "None of these examples is impressive, and I don't think it will interest the
  BoF audience."

The assistant then pitched an eBPF and Lua hybrid: TLS deep-packet inspection
in XDP, a minimal eBPF program matching TCP/443 and dispatching to kernel Lua
to walk the ClientHello and extract the SNI, the Lua side hot-swappable under
the live XDP program. The rebuttal:

- "I already presented a TLS parser in the main session. I presented Dome. I also
  don't want to use eBPF. You seem lazy to think: sloppy work, without
  understanding the essence of the BoF."

The assistant came back with a different direction, instrumenting the datapath
live while the bug happens, and two pitches in it:

- a deterministic fault injector: a Lua predicate to corrupt exactly the
  retransmitted segment that ACKs a chosen sequence number, edited by hand while
  narrowing down a protocol bug
- a live view of why packets die: a kprobe on `kfree_skb_reason()` printing the
  drop reason, edited on the spot to also dump socket state

- "I want you to follow this `kfree_skb_reason` path, I think that one is
  interesting."

## Building it

The assistant came back with two minimal commits, the probe patch and the
autogen spec, plus a first version of the tool on top of them:

- "Why is `SKB_CONSUMED` hardcoded?"
- "Let's validate the whole flow."
- "Did you validate the scripts?"
- "Is this stack trace yours? `[why: caught NO_SOCKET, registers: ...
  pc : kfree_skb_reason+0x0/0x1d8, lr : __udp4_lib_rcv+0x450/0x748]`"

## Shaping it for the stage

At this point the tool was called `why`, and it was two scripts plus a chardev.

- "Let's clean up the state and keep only this beat. I don't know why you kept
  the others. Let's save the previous state in a backup branch."
- "Can you explain the example to me?"
- "`driver.sentinel = setmetatable({}, {__gc = function() runner.stop(script)
  end})`: is this necessary? What happens if we don't use it?"
- "Let's organize the files. I want you to move the example to `examples`. Let's
  rename the scripts too: instead of `why` I want something more descriptive like
  `drop_reason`. Organize the house, clean the code as much as possible. I want
  to show it on stage in the cleanest way. I'll want to show the commits you
  generated too, adapting the patch and the autogen."
- "I prefer a single script instead of two, with a commented parameter for the
  watch case so we can modify it live."
- "I also want the example to have a usage comment to make the presentation
  easier, to be clear. Both pieces of code should have a comment at the top with
  a brief, self-contained explanation for the audience."
- "And I want you to validate everything, guarantee it will work on the day."
- "Why not use the env and print in the REPL?"

## The audit

The handler receives `argument(n)`, a closure that reads the probed function's
arguments out of `pt_regs`. The pointer it captures dies when the kprobe callback
returns:

- "Is the closure safe? Will nothing propagate/escape after the kprobe callback
  returns?"
- "Verify this possible-error flow deeply and guarantee it's possible; if so,
  apply the fix."

It found a real bug. When the Lua handler raised an error, the error object
shifted the stack, so the cleanup that nulls the captured pointer missed its
target and the pointer stayed dangling. Instrumenting the cleanup proved it
before the fix landed.

## The line-by-line review

- "`local WATCH -- = \"NO_SOCKET\"`: here the comment should be on the whole
  line, and above it there should be documentation of the possible values and
  what it does."
- "Why do you need a `local WATCH`? Why not remove that line and only uncomment
  it when necessary?"
- "`local name = {}; for k, v in pairs(reason) do name[v] = k end`: what do you
  need this for?"
- "`-- leave it nil to only count.`: you don't need that, right? If it's not
  defined, it'll be nil."
- "`local handlers = {pre = pre}` ... `probe.new(\"kfree_skb_reason\", handlers)`:
  you can use newlines to make it more comfortable to read."
- "In the `dropreason.lua` comment you have to put the command to simulate the
  drop too."

## The query API

Reading the counters started out as two REPL lines suggested in the example's
usage comment, and then as a function the caller handed the table back to:

- "`> t = {} require(\"rcu\").map(drops, function(k, v) ... end)`,
  `> table.sort(t) return table.concat(t, \"\\n\")`: put this in another script so
  that we can just require it and call a single function."
- "`return require(\"examples.dropreason_report\")(drops)`: instead of doing this,
  which is horrible, just put the code in a module with an intelligible name and
  a real API, something like `drops.report()`."

