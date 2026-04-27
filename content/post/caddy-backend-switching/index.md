---
title: Hot-swapping backends with Caddy and direnv
description: Using Caddy as a local reverse proxy to flip between a VM and a Azure backend, with direnv keeping things tidy
slug: caddy-backend-switching
date: 2026-04-27 00:00:00+1000
image: cover.png
categories:
  - Development
tags:
  - Caddy
  - direnv
  - Reverse Proxy
  - Local Dev
weight: 1
links:
  - title: Caddy
    description: Fast and extensible multi-platform HTTP/1-2-3 web server with automatic HTTPS
    website: https://caddyserver.com/
  - title: direnv
    description: Unclutter your .profile
    website: https://direnv.net/
---

## The problem

I work on a project where the frontend is a single-page app and the backend runs in a Windows VM via Parallels. There's also a deployed version of the backend hosted in Azure. Depending on what I'm working on, I need to hit one or the other—and switching between them was getting tedious.

The frontend needs HTTPS—things like camera access and mixed content rules were causing issues over plain HTTP in the local dev setup. The VM backend serves plain HTTP, so I needed something to put HTTPS in front of it.

On top of that, the API URL gets baked into the frontend bundle at build time. Changing it means restarting the dev server. So even if I _could_ just point at a different backend URL, I'd be restarting the dev server every time I switched.

I needed a local proxy that could serve HTTPS, sit at a stable URL the frontend never has to change, and let me swap the upstream behind it with minimal effort.

## Caddy as the gateway

[Caddy](https://caddyserver.com/) handles this perfectly. It automatically handles HTTPS certificates for `localhost`, so the frontend always talks HTTPS to `localhost:9443` and Caddy forwards traffic to whichever backend is active.

I set up two Caddyfiles—one pointing at the Azure environment and one pointing at the VM:

{{< code-hint "Caddyfile.azure" "~/project/Caddyfile.azure" >}}

```
{
	auto_https disable_redirects
}

https://localhost:9443 {
	reverse_proxy https://my-app.example.com {
		header_up Host {upstream_hostport}
	}

	header {
		Access-Control-Allow-Origin "http://localhost:3000"
		Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS"
		Access-Control-Allow-Headers "*"
		Access-Control-Allow-Credentials "true"
	}

	@options method OPTIONS
	respond @options 204
}
```

{{< code-hint "Caddyfile.vm" "~/project/Caddyfile.vm" >}}

```
{
	auto_https disable_redirects
}

https://localhost:9443 {
	reverse_proxy http://10.211.55.3:5000 {
		header_down -Access-Control-Allow-Origin
		header_down -Access-Control-Allow-Methods
		header_down -Access-Control-Allow-Headers
		header_down -Access-Control-Allow-Credentials
	}

	header {
		Access-Control-Allow-Origin "http://localhost:3000"
		Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS"
		Access-Control-Allow-Headers "*"
		Access-Control-Allow-Credentials "true"
	}

	@options method OPTIONS
	respond @options 204
}
```

The VM version strips any CORS headers the upstream might already set, so Caddy's own headers are the only ones the browser sees. The Azure version passes `Host` through so the upstream knows which tenant to serve.

Switching is one command:

```bash {linenos=false}
caddy reload --config ~/project/Caddyfile.azure
```

## direnv for scoped shortcuts

Typing that reload command every time is fine, but I wanted something snappier. I created two tiny scripts in a `.bin` directory:

{{< code-hint "caddy-azure" "~/project/.bin/caddy-azure" >}}

```bash
#!/bin/bash
CONFIG=~/project/Caddyfile.azure
caddy reload --config "$CONFIG" 2>/dev/null || caddy start --config "$CONFIG"
echo "→ Azure backend"
```

{{< code-hint "caddy-vm" "~/project/.bin/caddy-vm" >}}

```bash
#!/bin/bash
CONFIG=~/project/Caddyfile.vm
caddy reload --config "$CONFIG" 2>/dev/null || caddy start --config "$CONFIG"
echo "→ VM backend"
```

Now, I _could_ add `~/project/.bin` to my `PATH` in `.zshrc`. But that means every terminal session gets these project-specific commands on the path, even when I'm nowhere near this project. That's where [direnv](https://direnv.net/) comes in.

direnv watches for `.envrc` files as you `cd` around your filesystem. When you enter a directory that has one, it loads the environment. When you leave, it unloads it. One line in an `.envrc` is all it takes:

{{< code-hint ".envrc" "~/project/.envrc" >}}

```bash {linenos=false}
PATH_add .bin
```

From any terminal under `~/project`, I get `caddy-azure` and `caddy-vm` as commands. Step outside that tree and they disappear. No shell pollution, no remembering to clean up.

## The workflow

Day to day it looks like this:

1. `caddy-azure` — starts Caddy with the Azure config (or reloads if it's already running)
2. Start the frontend: `npm start`
3. Work against the Azure backend
4. Need to test against the VM? `caddy-vm`
5. Done with the VM? `caddy-azure`

The frontend doesn't care—it always hits `https://localhost:9443`. The proxy does the routing.

## One gotcha

**Clear your session after switching.** If the app caches auth tokens or session data (and it probably does), log out before switching backends. Stale tokens from one environment won't work against the other, and the errors aren't always obvious.
