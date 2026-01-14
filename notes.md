# Building Container Isolation in Rust
## Annotated Talk Draft with Review Notes

---

## Proposed Structure

| # | Section | Status |
|---|---------|--------|
| 1 | Intro - Who am I? | Solid hook |
| 2 | VMs vs. Containers | Needs cgroups decision |
| 3 | Rootless Containers | Demo decision pending |
| 4 | Why Rust? | Could combine with #5 |
| 5 | nix Crate | Could combine with #4 |
| 6 | Building a MVRC | Main event - needs flow work |
| 7 | [Optional] Compare to Go | Cut unless time permits |
| 8 | Bento Demo | Clarify: different from MVRC? |
| 9 | Kahoot | Cross bridge when we get there |
| 10 | Q&A | Standard |

---

## Section 1: Intro - Who am I?

### Your Draft
> SWE at Watahan in Tokyo. During my lunch break one day, I was thinking about the Docker environment, like I'm sure everyone here spends their lunch breaks. Staring at my bento lunch from Life, I started thinking about the meaning of containers and what went into them. I learned the basics of what containers are and decided to venture on a journey to learn to make my own. I was curious about Docker under the hood. Project is so cleverly called: bento.

### Notes
✅ **The bento story works.** It's personal, relatable, and explains the project name naturally.

⚠️ **Consider adding:** What's the takeaway for the audience? "By the end of this talk, you'll understand how containers actually isolate processes and have the tools to build one yourself."

⚠️ **Timing:** Keep this to ~2 minutes max. The story is good but don't let it stretch.

---

## Section 2: VMs vs. Containers (namespaces and cgroups)

### Your Draft
> Before starting this project, I assumed two things were the possible outcomes of running `docker run -it --name my-ubuntu-container ubuntu /bin/bash`:
> 1. I thought there were a bunch of VMs running on my mac
> 2. magic

**VMs:**
- hardware virtualization / machine virtualization
- VM isolated operating system (hypervisors)
- hypervisor = software to divvy up resources from the computer host (vcpu, vram, vnet)
- hypervisor → responsible for creating virtualized instances of each component

**Containers:**
- isolated processes `[!!REMINDER!! Don't forget to mention the OFFICE reference]`
- shared OS but think they are separate OS, independent from the rest
- concept isn't necessarily new, but their usage has become quite popular/standard
- operating system level virtualization

**Linux kernel specifics:**
- **namespaces** - restrict what the process can see `[!!!LIZRICEQUOTE!!!]`
- **cgroups** - resource control `[!!!ELABORATE!!!]`
  - pseudo-filesystem
  - usable resources

`[CLAUDE, show the namespaces for a process now or when doing the container?]`

`[Claude, is there a good way to show or demonstrate cgroups? Should I, if I'm not actually talking about cgroups in the talk?]`

### Notes
✅ **"VMs or magic" framing is good.** Honest and gets a laugh.

✅ **The VM vs Container breakdown is correct** and hits the key points.

❌ **DECISION NEEDED: cgroups.** You mention them here but explicitly skip them later. Pick one:
- **Option A:** Cut cgroups entirely. Say "Containers use namespaces for isolation and cgroups for resource limits. Today we're focusing on namespaces - cgroups is a whole separate talk."
- **Option B:** Give cgroups 60 seconds. Show `/sys/fs/cgroup/` briefly, explain it controls CPU/memory limits, move on.

I'd recommend **Option A** for a first talk. Cleaner scope.

❓ **Show namespaces here or during MVRC build?** 
→ **Recommendation:** Brief conceptual mention here ("namespaces are kernel features that restrict what a process can see"), then the actual `ls /proc/<pid>/ns` demo during the MVRC section when it's directly relevant.

❓ **What's the OFFICE reference?** Make sure you actually remember it during the talk - write out the actual line you want to say.

⚠️ **Linux-only disclaimer:** Good that you mention WSL2/Linux VM needed. Consider a slide with "Prerequisites: Linux kernel (WSL2, VM, or native Linux)" so people know upfront.

---

## Section 3: Rootless Containers

### Your Draft
> Containers are lightweight and portable process isolation on the host machine. Since they share the host kernel, additional security needs to be implemented to ensure proper isolation.
>
> By default, containers, for example, from Docker run as root from the host. Unless specified, they run as root which is the same root as on the host.
>
> Rootless Containers run with non-root users to mitigate potential vulnerabilities.
>
> Today we'll be building a MVRC (minimal viable rootless container) in Rust.

`[EXAMPLE - Claude, should I show this example or not? It may take too long and not super important especially if I have to wait for Docker to do shit.]`
> Run docker container, sleep 1000, then in another terminal `ps -eaf | grep sleep` and show it's as root (not container root). Mention you can run docker containers rootless, but aren't by default.

### Notes
✅ **The explanation is clear and correct.**

❓ **The Docker demo question:** This is actually important context. Without seeing *why* root-in-container = root-on-host is a problem, the audience might not care about rootless.

**Recommendations:**
1. **Pre-record it.** 30-second screen capture. Narrate live over the recording. Zero risk of Docker being slow.
2. **Or use a static slide** showing the terminal output of `ps -eaf | grep sleep` with arrows pointing to "UID 0 = root on host"

The point is: make the "why this matters" visual without live demo risk.

⚠️ **Typo:** "we'll bne building" → "we'll be building"

---

## Section 4: Why Rust?

### Your Draft
> I was looking for a project while going through the Rust book and thus began my journey on my project.
>
> I understood the project was going to be quite complicated so I thought no better reason than to facilitate this perceived complexity with the safety that Rust offers.
>
> There was the learning curve of course, but using Rust forced me to think deeply before implementation. Even though I struggled with error handling, I was glad it was there.

**youki quote:**
> "Rust is one of the best languages to implement the oci-runtime spec. Many very nice container tools are currently written in Go. However, the container runtime requires the use of system calls, which requires a bit of special handling when implemented in Go. This is tricky (e.g. namespaces(7), fork(2)); with Rust too, but it's not that tricky. And, unlike in C, Rust provides the benefit of memory safety."

`[!!REMINDER!! Find an example here to demonstrate this.]`
`[!!!OPTIONAL!!! Reference Go example later.]`

### Notes
✅ **Honest framing:** "I was learning Rust and wanted a real project" is relatable.

⚠️ **Consider combining with Section 5 (nix crate).** They're both "why Rust/how Rust" and together they're maybe 3-4 minutes. Keeps momentum.

❓ **Go comparison:** You have it as optional in Section 7 and mentioned here. Make one decision:
- If you're going to show Go's fork complexity, do it HERE as part of "why Rust" (2 slides max)
- Or cut it entirely

The youki quote already makes the point. Showing Go code might be overkill unless you have a really clean example ready.

⚠️ **"I struggled with error handling"** - this is honest but be careful about framing. You could say: "The compiler forced me to handle errors I would have ignored in other languages - frustrating at first, but exactly what you want when you're messing with system calls."

---

## Section 5: nix Crate

### Your Draft
> My BFF crate is nix. It provides a safe alternative to unsafe APIs that are exposed by the libc crate.
>
> "Nix provides a safe alternative to the unsafe APIs exposed by the libc crate. This is done by wrapping the libc functionality with types/abstractions that enforce legal/safe usage."

```rust
// libc api (unsafe, requires handling return code/errno)
pub unsafe extern fn gethostname(name: *mut c_char, len: size_t) -> c_int;

// nix api (returns a nix::Result<OsString>)
pub fn gethostname() -> Result<OsString>;
```

### Notes
✅ **This example is perfect.** Clear, concrete, immediately shows the value.

✅ **"My BFF crate"** - good personality, keeps it light.

💡 **Suggestion:** Combine Sections 4 and 5 into "Why Rust + The Tools We'll Use" (~3-4 min total). Flow:
1. "I wanted a challenging project while learning Rust"
2. "Turns out Rust is great for this - here's what youki says..."
3. "The nix crate is what makes this ergonomic" + show the comparison

---

## Section 6: Building a MVRC (minimal viable rootless container)

### Your Draft

**Goal:**
> Similar to: `docker run <image> <command> <argv>`
> We'll do: `cargo run -- <command> <argv>`

**Namespaces to cover:**
- user - user and group IDs isolation
- mount - mounts seen by the process
- pid - isolates the process ID
- uts - isolates hostname of process
- (mention but skip: network, ipc)

**Syscalls to use:**
- `unshare()` - control shared execution context
- `mount/umount()` - attach/detach filesystem
- `execv` - replace current process image
- `fork()` - duplicate calling process
- `chroot()` - change root directory

**Build flow (under construction):**
1. Build test CLI
2. Show process namespaces via `/proc/<pid>/ns`
3. Use nix crate, add getpid
4. Unshare PID namespace (still sudo)
5. Show PID is 1 inside container
6. Try UTS - realize we need sudo
7. Explain UID/GID mapping for rootless
8. "We rootless, baby!"
9. Unshare mount namespace
10. Use pre-extracted busybox rootfs
11. chroot + change working dir
12. Demo: inside thinks it's root, can't access host FS
13. Show PID from inside vs outside

### Notes
✅ **This is the main event.** The flow makes sense conceptually.

⚠️ **"Flow under construction"** - this is where you need to invest time. Suggestions:

**Structure the live coding more explicitly:**

| Step | What you type/show | What audience learns |
|------|-------------------|---------------------|
| 1 | `sleep 1000 &` then `ls /proc/<pid>/ns` | "Every process has namespaces" |
| 2 | Show your Cargo.toml with nix | "Here's our starting point" |
| 3 | Basic CLI that calls getpid() | "We can call syscalls from Rust" |
| 4 | Add unshare(CLONE_NEWPID) | "Now we isolate PID namespace" |
| 5 | Run it - PID is 1! | "The process thinks it's init" |
| ... | etc | ... |

**Write out the exact commands and code snippets** you'll type/show. No improvising during live coding.

❓ **Do you have the busybox rootfs ready?** Make sure this is pre-downloaded and tested before the talk.

⚠️ **Namespace list:** You're missing `cgroup` namespace (different from cgroups the feature). That's fine to skip, but be aware someone might ask.

⚠️ **The `/proc` explanation** - you have it marked as "old but still usable" and "rewrite from manpages." For the talk, you just need one sentence: "/proc is a pseudo-filesystem that exposes kernel data structures - we'll use it to see our process's namespaces."

---

## Section 7: [Optional] Compare to Go

### Notes
❌ **Recommendation: Cut this.** 

You already mention Go in Section 4 via the youki quote. Actually showing Go code is:
- Extra prep work
- Risk of spending time on something tangential
- Potentially alienating if Go folks in audience

If someone asks "why not Go?" in Q&A, you have the youki quote ready.

---

## Section 8: Bento Demo

### Your Draft
(No content provided)

### Notes
❓ **Key question: How is this different from the MVRC you just built?**

Options:
1. **MVRC = minimal version, Bento = full version.** After the teaching section, show "here's what I built on top of this - OCI image parsing, daemon mode, proper lifecycle management." (This is probably what you mean?)

2. **They're the same thing.** In which case, merge this with Section 6 and just end with a polished demo.

**Suggestion for Option 1:**
- 2-3 minutes max
- Show the real `bento` CLI: `bento create`, `bento start`, `bento exec`
- "Everything we built today is the foundation - here's what you can build on top"
- Maybe show one thing the audience didn't build: OCI image parsing or overlay filesystem

---

## Section 9: Kahoot

### Notes
You said we'll cross this bridge later. Fair.

**If you keep it:**
- 5 questions max
- Quiz on concepts from the talk (namespaces, syscalls, rootless)
- Have a backup plan if wifi fails

**If you cut it:**
- More time for Q&A
- Less risk
- First talk = simpler is better

---

## Section 10: Q&A

### Notes
✅ Standard. 

**Prep for likely questions:**
- "Why not just use Docker?" → Learning exercise, understanding internals
- "How does this compare to youki/runc?" → Much simpler, educational, not production
- "What about cgroups?" → Scope for another talk, namespaces first
- "Can this run on Mac?" → No, Linux kernel only, need WSL2/VM/native Linux
- "What's next for bento?" → OCI spec compliance, cgroups, maybe network namespace

---

## Open Questions to Resolve

| Question | Your Options | Recommendation |
|----------|--------------|----------------|
| Show cgroups? | Explain briefly / Cut entirely | Cut - cleaner scope |
| Rootless Docker demo? | Live / Pre-recorded / Static slide | Pre-record or static |
| Show Go comparison? | In Section 4 / Separate section / Cut | Cut |
| MVRC vs Bento demo? | Same thing / Different scope | Clarify - probably different scope |
| Kahoot? | Keep / Cut | Decide after timing a practice run |
| Time slot? | Unknown | FIND OUT - drives all other decisions |

---

## Suggested Revised Structure

| # | Section | Time Est. |
|---|---------|-----------|
| 1 | Intro - Who am I, the bento story | 2 min |
| 2 | VMs vs Containers (namespaces focus, cgroups acknowledged but skipped) | 5 min |
| 3 | Why Rootless Matters (pre-recorded demo or slide) | 3 min |
| 4 | Why Rust + nix crate (combined) | 4 min |
| 5 | Building the MVRC (main event, live coding) | 20 min |
| 6 | Full Bento Demo (show what you built beyond MVRC) | 5 min |
| 7 | Q&A | remaining |
| — | Kahoot (optional, if time/energy) | 5-10 min |

**Total without Kahoot:** ~40 min + Q&A
**Total with Kahoot:** ~50 min + Q&A

---

## Next Steps

1. **Find out your time slot.** Everything else depends on this.
2. **Write out the exact MVRC build steps** with code snippets you'll show.
3. **Decide on the cgroups/Go/Kahoot cuts** based on timing.
4. **Practice once end-to-end** with a timer. See where you actually land.
5. **Pre-record the rootless Docker demo** as a safety net.
