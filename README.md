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

11. Intro? Who am I?

- SWE at Watahan in Tokyo. During my lunch break one day, I was thinking about the Docker environment, like I'm sure everyone here spends their lunch breaks. Staring at my bento lunch from Life, I started thinking about the meaning of containers and what when into them. I learned the basics of what containers are and decided to venture on a journey to learn to make my own. I was curious about Docker under the hood. Project is so cleverly called: bento.

2. VMs vs. Containers (namespaces and cgroups)

Before starting this project, I assumed two things were the possible outcomes of running
`docker run -it --name my-ubuntu-container ubuntu /bin/bash`

1. I thought their were a bunch of vms running on my mac
2. magic

I first started with VM vs Container

**Containers** vs **VMs**: What's the difference?: https://www.youtube.com/watch?v=cjXI-yxqGTI

**vms**:

- hardware virtualization
  machine vitualization
- vm isolated operating system (hypervisors)
- hypervisor = software to divy up resources from the computer host
  vcpu, vram, vnet

-> hypervisor = responsible for creating these virtualized instances of each component that make the machines
-> hardware

containers:

- isolated processes [!!REMINDER!!] Dont forget to mention the OFFICE reference
- shared os but think they are separate os, independent from the rest.

concept isn't necessarily new, but their usage has become quite popular or standard -
operating system level virtualization
-> containers
-> host os: hosts the containers
-> kernel
-> hardware

linux kernel (specific, explain that a linux vm, container, linux machine, or wsl2 are needed to follow along as these topics are specific to the linux kernel) -
**namespaces**

- namespaces restrict what the process can see [!!!LIZRICEQUOTE!!!]
  set using kernel syscalls
  [CLAUDE, show the namespaces for a process now or when doing the container?]

**cgroups** = resource control [!!!ELABORATE!!!]
What resources the process can use.

- pseudo-filesystem
- usable resources
  [Claude, is there a good way to show or demonstrate this? Should I, if I'm not actually talking about the cgroups in the talk.]

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
