**Talk Structure: "Building Container Isolation in Rust"**

1. Intro? Who am I?
2. VMs vs. Containers (namespaces and cgroups)
3. Rootless Containers
4. Why Rust?
5. nix Crate
6. Building a MVRC (minimal viable rootless container)
7. [Optional] Compare to GO
8. Bento Demo
9. Kahoot
10. Q&A

## Section 1: Intro - Who am I?

Hi, Rustaceans. My name is Carlo Quick and I am a SWE at Watahan in Tokyo. Like many of you, I sometimes spend my lunch breaks thinking about Docker containers. So one day I'm sitting in the park outside work, eating a bento - if you've never had one, they're these pre-packaged meals you can get almost anywhere in Japan - and I realize I have no idea what a container actually is under the hood. (On this one particular lunch break, I found myself staring at my bento container wondering what Docker does under the hood.)

I set out to learn the basics and quickly became very interested. I don't want to get ahead of myself. We'll visit what became of my lunch later in the talk.

Today, we are going to build a Minimal Viable Rootless Container in Rust. By the end of this talk, you'll understand how containers actually isolate processes and have the tools to build one yourself.

So let's get started by understanding the differences between virtual machines and containers.

## Section 2: VMs vs. Containers (namespaces and cgroups)

Before starting this project, I assumed two things were the possible outcomes of running
`docker run -it --name busybox-container busybox /bin/sh`

1. There were a bunch of VMs running on my local machine, or
2. Magic

(I could imagine little binary Harry Potters waving a wand and saying "Creatus Busyboxus Continens" and poof, I had busybox up and running)

First, what's a container and how do they compare to Virtual Machines?

**VMs:**

- Hardware/machine virtualization
- Each VM is an isolated operating system
- Hypervisor divvies up host resources (vCPU, vRAM, vNet)
- Stack: Hypervisor → Hardware

**Containers:**

- Isolated processes
- Shared OS kernel, but each container thinks it's independent
- Not a new concept, but usage has become standard
- Stack: Containers → Host OS → Kernel → Hardware

**How does Linux pull this off?**

- **Namespaces** - regulate what a process can see or access
- **Cgroups** - control what resources a process can use (CPU, memory, PIDs, etc.)

**The Office Analogy:**

In the Office, there's an episode where Michael Scott ventures into the wilderness thinking he's surviving alone. What he doesn't know is Dwight is watching the entire time, ready to intervene. Michael thinks he's independent, but Dwight sees everything and controls whether Michael survives or not.

That's container isolation. The container thinks it's its own OS with PID 1 and its own root filesystem. It can't see the host, but the host sees everything and can even shut it down whenever it wants.

---

Containers use namespaces for isolation and cgroups for resource limits. Today we're focusing on namespaces - cgroups is a whole separate talk.

Now let's talk about going rootless.

---
Resources:
What's the difference?: https://www.youtube.com/watch?v=cjXI-yxqGTI

3. Rootless Containers

containers are lightweight and portable process isolation on the host machine. Since they share are host kernel, additional security needs to be implemented to ensure proper isolation.

By default, containers, for example, from Docker run as root from the host. Unless specified, they run as root which is the same root as on the host.

Rootless Containers run with non-root users to mitigate potntial vulnerabilities.

Today we'll bne building a MVRC (minimal viable rootless container) in Rust.

[EXAMPLE]
[Claude, should I show this example or not? It may take too long and not super important especially if I have to wait for Docker to do shit.]
run docker container, sleep 1000, then in another terminal ps -eaf | grep sleep and show it's as root (not container root). Mention you can run docker containers rootles, but arent be default.

4. Why Rust?

I was looking for a project while going through the Rust book and thus began my journey on my project.

I understood the project was going to be quite complicated so I thought no better reason than to fascilitate this preciecd complexity with the safety that rust offer.

There was the learning curve of course, but using rust forced me to think deeply before implementation. Even though I struggled with error handling, I was glad it was there.

I don't have an current metrics but I can refer you to the other major player in the Rust containization field - youki.
[picture and description of youki metrics and inspiration to use Rust]

From youki:
Here is why we are writing a new container runtime in Rust.

    Rust is one of the best languages to implement the oci-runtime spec. Many very nice container tools are currently written in Go. However, the container runtime requires the use of system calls, which requires a bit of special handling when implemented in Go. This is tricky (e.g. namespaces(7), fork(2)); with Rust too, but it's not that tricky. And, unlike in C, Rust provides the benefit of memory safety. While Rust is not yet a major player in the container field, it has the potential to contribute a lot: something this project attempts to exemplify.

[!!REMINDER!!] Find an example here to demonstrate this. [!!!OPTIONAL!!!] Reference Go example later.

5. nix Crate
   My bff crate is nix. It provides a safe alternative to unsafe APIs that are exposed by the libc crate.

"Nix provides a safe alternative to the unsafe APIs exposed by the libc crate. This is done by wrapping the libc functionality with types/abstractions that enforce legal/safe usage." - https://docs.rs/crate/nix/latest

[EXAMPLEFROMNIX]

```rust
// libc api (unsafe, requires handling return code/errno)
pub unsafe extern fn gethostname(name: *mut c_char, len: size_t) -> c_int;

// nix api (returns a nix::Result<OsString>)
pub fn gethostname() -> Result<OsString>;
```

6. Building a MVRC (minimal viable rootless container)

Alright, it's time to start coding ourselves a MvRC. We'll be focusing more today on using namespaces to get process isolation and will save cgroups for later.

Similar to:
docker run <image> <command> <argv>
cargo run -- <command> <argv>

<CodePortion  />
[build the test cli!]

<TerminalPortion />
[make a process sleep then show its pid, then its namespaces]

**namespaces** allow or this illusion isolation [!!!ELABORATE!!!]
[example] make a process sleep, then in another terminal run ps -eaf | grep sleep, find tht pid, then ls /proc/<pid>/ns and show the namespaces available to the process

//\***\* Old but still usable
**/proc** [!!REWRITE_FROM_MANPAGES!!]
The proc filesystem is a pseudo-filesystem which provides an
interface to kernel data structures. It is commonly mounted at
`/proc`. Typically, it is mounted automatically by the system, but
it can also be mounted manually using a command such as:
** //

- namespaces restrict and limit what the process can see [!!!LIZRICE!!!]
  set using kernel syscalls

[Detail what each namespace is and does]
namespaces:
[Claude, am I missing anything?]

- user - user and group ids isolation
- mount - mounts seen by the process
- pid - isolates the process id
- uts - isolates hostname of process
  --- not covered in this talk but equally important --
  network - isolation of networking system resources
  ipc - "isolate between different IPC resources like message queues, semaphores and shared memory." - Shlomi Boutnaru, Ph.D

We can cause affect to the process namespaces using syscalls (system calls) which are APIs provided by the Linux Kernel
syscalls:

- unshare(): "allow a process to control its shared execution context" - unshare(2)
- mount/umount(): attaches or detaches a filesystem
- execv - replaces current process image with another (not a new process)
- fork():
  "creates a new process by duplicating the calling process.
  The new process is referred to as the child process. The calling
  process is referred to as the parent process." - fork(2)
- chroot(): Changes the processes root directory of the process

references:
https://man7.org/linux/man-pages/man7/namespaces.7.html
https://man7.org/linux/man-pages/man7/cgroups.7.html

https://man7.org/linux/man-pages/man2/unshare.2.html
https://man7.org/linux/man-pages/man2/umount.2.html
https://man7.org/linux/man-pages/man8/mount.8.html
https://man7.org/linux/man-pages/man2/execve.2.html
https://man7.org/linux/man-pages/man2/fork.2.html
https://man7.org/linux/man-pages/man2/chroot.2.html

<CodePortion  /> //**FLOW UNDER CONSTRUCTION**//
[Use the nix crate to add getpid and display it in rust]

[still using sudo]
unshare pid and show the process id before and after. If the child is 1, we've made a very simple container by isolating the process

[Try and change the UTS and you'll see we need to be sudo. Show that we can run it using sudo, but that's not what we want.]

[explain the uid and gid mapping. to go rootless and now we can run the container as its own root]

[We rootless, baby!]
[unshare the mount ns]

[explain that there is a already extracted, busybox rootfs from docker hub]

[chroot, and change current working dir]

[demonstrate it thinks it's root and the fs it has. try running something from outside the container and you cant. make it sleep. show the pid, then from outside the container the pid it really is from the parent perspective]

7. Kahoot
8. Bento Demo
9. Q&A
