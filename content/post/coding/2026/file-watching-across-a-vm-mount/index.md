---
title: "File Watching Doesn't Cross the Mount: Why My Code Lives in the VM"
date: 2026-11-29T14:36:01+11:00
publishDate: 2026-11-29
draft: false
layout: "post"
tags: ["lima", "devvm", "macos", "linux", "inotify", "virtiofs", "vite", "file-watching"]
categories: ["coding"]
description: "Why sharing a folder from macOS into a Linux VM breaks the file watchers that dev servers, test runners and IDEs depend on, and why I checked out every project inside the VM instead."
author: "Paul"
---

When I moved my development into a [Linux VM on my Mac](/pauls-blog/post/coding/2026/one-vm-instead-of-seven-devcontainers/), the obvious setup was to keep the code on the Mac and share the folder into the VM. My editor and Finder would see the files as normal, Time Machine would back them up, and the VM would just run things.

I didn't do that. Every project is checked out inside the VM, on the VM's own disk, and nothing from the Mac is mounted:

```yaml
mounts: []
```

`node_modules` was one reason, which I covered in the earlier post. The other is file watching. Dev servers, test runners and IDEs all rely on the operating system telling them when a file changes, and that notification doesn't reliably make it from macOS into a Linux VM.

## How watching works

A watcher doesn't read your files over and over. It asks the kernel to tell it when something changes. On Linux that's **inotify**. On macOS it's **FSEvents**. Vite, webpack, Jest's watch mode, `tsc --watch`, nodemon, Hugo's server and the IntelliJ indexer all sit on top of one or the other.

The important part is *where* the event comes from. inotify fires when the Linux kernel itself handles a change: a process in the VM writes a file, the kernel updates it, and anything watching that file hears about it.

## What breaks across a shared folder

Now share a folder from the Mac into the VM, with virtiofs, 9p or sshfs (the options Lima offers). Edit a file on the Mac, and the write happens in macOS. It goes to APFS, and macOS's FSEvents hears about it. The VM's kernel didn't do the write, so it has nothing to raise an inotify event for. The file has changed when the VM next reads it, but nobody told the watcher to read it.

So the setup I almost had was:

{{< mermaid >}}
flowchart LR
    E[Editor on the Mac] -->|write| A[APFS]
    A -->|FSEvents| M[Mac watchers: fine]
    A -->|virtiofs share| V[VM sees new contents<br/>on next read]
    V -.->|no inotify event| D[Vite / Jest / tsc --watch in the VM]
{{< /mermaid >}}

Edit on the Mac, the dev server in the VM never reloads. That's the worst direction to break, because it's the normal way of working: editor on one side, running app on the other.

Some layers try to patch this by forwarding host events into the guest. Docker Desktop does it for bind mounts, and Lima has an experimental option for it. How well that works varies with the mount type, the tool and how many files change at once. A `git checkout` that touches hundreds of files is a very different load from saving one. I didn't want to depend on it.

## Polling isn't a fix

The usual workaround is to stop asking the kernel and poll instead: `server.watch.usePolling` in Vite, `CHOKIDAR_USEPOLLING=true` for chokidar-based tools, `watchOptions.poll` in webpack. The watcher re-checks every file's timestamp on a timer.

That works, but it's a poor trade:

- **It's slow to notice.** A change shows up on the next poll, not straight away.
- **It costs CPU all the time**, even when nothing changes.
- **It scales with file count.** A JavaScript project with `node_modules` has a lot of files, and polling across a network-style filesystem makes every check slower again.
- **It's per tool.** Every watcher has its own polling switch, and each project has to remember to set it. Miss one, and that tool quietly stops reacting.

IDEs show the same problem in their own way. When the IntelliJ platform can't watch files, it falls back to rescanning directory trees, which looks like an IDE that's always indexing. JetBrains also says plainly that it doesn't support network filesystems for its backend and wants local disk.

## Putting everything on one side

The fix was to stop splitting the work across the boundary. The code, the dev servers, the tests, the agent and the IDE backend all live in the VM, on its ext4 disk:

```text
macOS host (thin)                      Lima VM
├── Claude Code desktop app ───SSH───► ├── ~/projects/current/…  every repo
├── JetBrains Gateway ───────SSH───►   ├── WebStorm backend
├── Browser → hugo.devvm:8000, …       ├── dev servers, tests, Docker
```

Now every write happens inside the VM, whoever makes it:

- **Claude Code** edits files over SSH, so its writes are the VM kernel's writes.
- **The IDE backend** runs in the VM. JetBrains Gateway (or VS Code's Remote-SSH) puts only the UI on the Mac, and the edits happen on the VM's disk.
- **`git checkout`, `pnpm install`, code generators** all run in the VM.

The kernel sees every change, inotify fires every time, and no tool needs a polling setting. Vite reloads, Jest re-runs, Hugo rebuilds the blog preview.

There's one more setting that matters once the watchers work. Linux limits how many files each user can watch, and the default is far too low for an IDE watching dozens of repos. Past the limit, the IntelliJ platform quietly stops watching and goes back to rescanning. So the VM raises it to JetBrains' recommended value on every boot, and a startup check makes sure it took:

```bash
printf 'fs.inotify.max_user_watches = %s\nfs.inotify.max_user_instances = %s\n' \
  524288 512 > /etc/sysctl.d/60-jetbrains-inotify.conf
sysctl -p /etc/sysctl.d/60-jetbrains-inotify.conf
```

That's a kernel setting, which is one more reason for a VM over a container: in Docker Desktop, the kernel belongs to Docker's VM, not to you.

## What I gave up

Keeping the files only in the VM has real costs:

- **No Finder, no Time Machine.** The VM's disk is excluded from backups, so code is only safe once it's pushed. Pushing before I stop for the day is an obligation now.
- **Moving files takes a step.** `limactl copy`, git, or VS Code's file explorer over SSH.
- **Xcode stays on the Mac**, so iOS work does a round trip.

For the work I do (Node and pnpm, dev servers with hot reload, IDE backends, an agent that edits and runs tests all day), working file watchers are worth more than seeing the files in Finder. The rule is simple: the code lives where it runs, and so does everything that edits it.
