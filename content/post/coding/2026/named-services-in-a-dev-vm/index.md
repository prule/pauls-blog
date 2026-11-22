---
title: "Names Instead of Port Numbers: Caddy in My Dev VM"
date: 2026-11-22T16:00:00+11:00
publishDate: 2026-11-22
draft: true
layout: "post"
tags: ["caddy", "lima", "devvm", "networking", "vite", "hugo", "reverse-proxy"]
categories: ["coding"]
description: "How every web service and project in my dev VM got a name like hugo.devvm:8000 through a Caddy reverse proxy, one port forward and a few /etc/hosts lines, and the three things that broke along the way."
author: "Paul"
---

My [dev VM](https://github.com/prule/devvm) runs a lot of web servers: the blog preview, SonarQube, and a Vite dev server for each of several projects. Each had its own port, and I had to remember which was which. Was `rss-reader` on 5182 or 5183?

Now each one has a name, all on the same port:

| Name | Goes to (VM port) |
|---|---|
| http://hugo.devvm:8000 | 1313, the blog preview |
| http://sonar.devvm:8000 | 9000, SonarQube |
| http://rss-reader.devvm:8000 | 5182 |
| http://leave-planner.devvm:8000 | 5185 |
| … | … |

It takes three pieces: Caddy in the VM, one port forward, and a line in the Mac's `/etc/hosts`.

{{< mermaid >}}
flowchart LR
    B[Browser on the Mac<br/>http://hugo.devvm:8000] -->|/etc/hosts: 127.0.0.1| F[Mac port 8000]
    F -->|Lima port forward| C[Caddy on VM port 80]
    C -->|Host: hugo.devvm| H[localhost:1313 Hugo]
    C -->|Host: sonar.devvm| S[localhost:9000 SonarQube]
    C -->|Host: rss-reader.devvm| R[localhost:5182 Vite]
{{< /mermaid >}}

## Caddy routes by name

Caddy runs inside the VM, listening on port 80 on the VM's localhost. It reads the `Host` header and passes the request to that name's port:

```text
{
	default_bind 127.0.0.1
}
http://hugo.devvm {
	redir / /pauls-blog/
	reverse_proxy localhost:1313
}
http://sonar.devvm {
	reverse_proxy localhost:9000
}
http://rss-reader.devvm {
	reverse_proxy localhost:5182
}
```

The `http://` prefix on each name tells Caddy to serve plain HTTP. Without it, Caddy would try to get certificates, and I'd have a local CA to trust on the Mac. For a dev VM on localhost, plain HTTP is fine.

The upstreams are `localhost`, not `127.0.0.1`, on purpose. Vite, unless told otherwise, may listen on the IPv6 `::1` only. `localhost` lets Caddy try both addresses.

The Caddyfile lives in `dev.yaml` like everything else, and every boot rewrites `/etc/caddy/Caddyfile` from it.

The big win is that Caddy is *inside* the VM, so it can reach any VM port. Adding a project is one block in the Caddyfile. No new port forward.

## One forward, to port 8000

The plan was `http://hugo.devvm`, no port at all. That meant forwarding the Mac's port 80 to Caddy. But Lima didn't forward the Mac's port 80: it's a privileged port, and the names just didn't load.

So the forward goes to the Mac's port 8000 instead:

```yaml
portForwards:
  - guestIP: "127.0.0.1"
    guestPortRange: [80, 80]
    hostIP: "127.0.0.1"
    hostPortRange: [8000, 8000]
```

The names carry `:8000` now. The Caddyfile didn't have to change, because Caddy matches the name and ignores the port in the `Host` header. One fixed port for everything is still much better than a port per service.

## The Mac has to know the names

`hugo.devvm` doesn't resolve anywhere by itself, so each name goes in the Mac's `/etc/hosts`, pointing at `127.0.0.1`. I didn't want to keep that list in step by hand, so the host-side bootstrap script reads the names straight out of `dev.yaml`'s Caddyfile and adds any that are missing:

```bash
missing=""
for name in $(grep -oE 'http://[a-z0-9-]+\.devvm' "$HERE/dev.yaml" | sed 's|http://||' | sort -u); do
  awk -v n="$name" '$1=="127.0.0.1" { for (i=2; i<=NF; i++) if ($i==n) f=1 } END { exit !f }' /etc/hosts \
    || missing="$missing $name"
done
if [ -n "$missing" ]; then
  echo "127.0.0.1$missing" | sudo tee -a /etc/hosts > /dev/null
fi
```

It only asks for my sudo password when there's a name to add. The Caddyfile is the one list, and `/etc/hosts` follows it.

## Each project owns its port

For a name to mean anything, the port behind it has to be fixed. Vite's habit of moving to the next free port when its port is taken would break that silently: the name would go to whatever happened to be on 5182. So each project pins its own port and fails when it's taken:

```ts
server: {
  port: 5186,
  strictPort: true,
  allowedHosts: ['.devvm'], // Vite refuses unknown Host headers (403) without this
},
```

`allowedHosts` is the other half. Recent Vite versions reject requests whose `Host` header they don't recognise, as protection against DNS rebinding. Behind Caddy, every request arrives as `something.devvm`, so Vite needs telling that's expected.

New projects take the next port up from 5186, and the README's table of names is the register of which project has which.

## Servers that know their own address

Some servers build absolute links or open a websocket back to themselves. Hugo does both: every link uses the `baseURL`, and live reload opens a websocket. Run it plainly, and the links point at `localhost:1313`, which bypasses Caddy.

So the blog preview tells Hugo its public address:

```bash
exec hugo server \
  --buildDrafts \
  --bind 127.0.0.1 --port 1313 \
  --baseURL http://hugo.devvm:8000/pauls-blog/ --appendPort=false \
  --liveReloadPort 8000 \
  --disableFastRender
```

`--appendPort=false` stops Hugo adding `:1313` to the base URL, and `--liveReloadPort 8000` points the live-reload websocket at the Mac's end of the forward. Links and live reload both go through Caddy now.

## The one that hung everything

After all that, every name hung. No error, no 502, just nothing.

VS Code was the cause. I'd connected VS Code's Remote-SSH to the VM, and it auto-forwards ports it sees in use. It had grabbed the Mac's port 8000 for a forward of its own, so Lima couldn't take it. Lima already forwards the VM's ports, so VS Code's forwarding was only getting in the way. The VM now turns it off on every boot, in VS Code's remote settings:

```json
"remote.autoForwardPorts": false
```

If every name hangs, something on the Mac holds port 8000. `sudo lsof -nP -iTCP:8000 -sTCP:LISTEN` on the Mac shows what. Free the port and restart the VM, because Lima only sets up its forwards at boot.

## Reading the errors

Once it's set up, the failures tell you where to look:

- **`502 Bad Gateway`**: Caddy is fine, but nothing is listening on that port. Start the project's dev server.
- **The browser can't find the name**: it's missing from the Mac's `/etc/hosts`.
- **Every name hangs**: something else holds the Mac's port 8000.
- **`403` from a Vite project**: `allowedHosts` is missing.

The old `localhost:PORT` addresses still work, too. The names are just easier to remember.
