---
title: tuns
description: Host public web services on localhost using SSH
keywords: [pico, tuns]
toc: 1
---

> NOTICE: This is a premium [pico+](/plus) service

# Features

- A zero-install developer tool using `ssh`
- Host public web services on `localhost`
- Host public tcp services on `localhost`
- Automatic TLS
- Custom domains
- Alerting for tunnel connect/disconnects
- Per-site [analytics](/analytics)
- [Multi-region support](/regions)
- Share your local webserver privately with another user

Using SSH tunnels, we can forward requests to your localhost from https, wss,
and tcp.

![Eric connects to sish on the Internet with the command 'ssh -R eric:80:localhost:3000 tuns.sh'. Tony visits 'https://eric.tuns.sh', which connects to sish, and forwards Eric's local server to Tony.](https://docs.ssi.sh/hiw-sish-public.png)

## Demo

<iframe width="100%" height="315" src="https://www.youtube.com/embed/wZHiuR9RqGw?si=AJLBbg5jc8ET0lvB" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

> [tuns demo](https://www.youtube.com/watch?v=wZHiuR9RqGw)

# Use cases

Think of tuns as a developer tool. It is designed for the individual developer
who is looking to prototype, demo, or otherwise deploy services without the
overhead of managing a production environment with TLS, HTTP reverse proxy, and
provisioning cloud infrastructure.

By using tuns you get automatic and public https for local web services.

Want to prototype a web service without fully deploying it in the cloud? You can
go from starting a local web service to sharing it with the world using a single
SSH command.

Hosting public web services from your home has never been easier with tuns.

# Docs

We manage a completely separate doc site for all things related to `sish`:

- [Read the sish docs](https://docs.ssi.sh)

# Example Usage

```bash
# if you have a local webserver on localhost:8000:
ssh -R dev:80:localhost:8000 tuns.sh
# now anyone can access it at https://{user}-dev.tuns.sh
```

# TUI

We have a TUI viewer to you can see all your active tunnels for monitoring.

```bash
ssh pico.sh
# -> tuns
```

[![tuns tui](/tuns-tui.png)](/tuns-tui.png)

# Alerts

We provide notifications for connect and disconnect events using our pico+ RSS
feed.

You can also see the alerts into our tuns TUI.

# Regions

> tuns.sh is a global service!

See our [regions page](/regions) to learn more about our geographical service
coverage.

# User namespace

When creating a tunnel to `tuns` we always prefix the name with your username:

```
{user}-{name}.tuns.sh
```

This includes when a client is using tuns as a ProxyJump:

```bash
ssh -R foobar:22:localhost:22 tuns.sh
# On the client side
ssh -J tuns.sh {user}-foobar
```

# Custom Domains

We require you to set up `CNAME` and `TXT` records for the domain/subdomain you
would like to use for your forwarded connection. The `CNAME` record must point
to `tuns.sh`. The TXT record name must be `_sish.customdomain` and contain the
SSH key fingerprint used for creating the tunnel. This key must also be linked
to your pico+ account.

You can retrieve your key fingerprint by running:

```
ssh-keygen -lf ~/.ssh/id_rsa | awk '{print $2}'
```

Example:

```
customdomain.example.com.          300     IN      CNAME   tuns.sh.
_sish.customdomain.example.com     300     IN      TXT     "SHA256:mVPwvezndPv/ARoIadVY98vAC0g+P/5633yTC4d/wXE"
```

Once set up, you can then create tunnels via your custom domain like this:

```
ssh -R customdomain.example.com:80:localhost:8000 tuns.sh
```

Make sure to select the correct (closest) tuns instance. You can find the
instance you're connected to from the connection output from tuns:

```
You are connected to tuns.sh. The following tunnels are only accessible on this instance.
```

In this case, my CNAME would use `tuns.sh`

You may want to pre-select the region you connect to. Try pinging `ash.tuns.sh`
or `nue.tuns.sh` to find the instance closest to you (lowest latency), and use
that for both your SSH connection and CNAME.

## Debug custom domains

First check the main record:

```bash
dig customdomain.example.com

; <<>> DiG 9.18.36 <<>> customdomain.example.com
;; QUESTION SECTION:
;customdomain.example.com.               IN      A

;; ANSWER SECTION:
customdomain.example.com.        60      IN      A       141.148.85.124
```

Then check the `TXT` record:

```bash
dig -t txt +short _sish.customdomain.example.com

SHA256:mVPwvezndPv/ARoIadVY98vAC0g+P/5633yTC4d/wXE
```

# tunmgr

A tunnel manager for docker services.

tunmgr automatically set's up tunnels for docker services. It utilizes `Expose`
ports as well as DNSNames (and the container name/id) to setup different
permutations of tunnels.

> [source code](https://github.com/picosh/tunmgr)

```yml
services:
  tunmgr:
    image: ghcr.io/picosh/tunmgr:latest
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - $HOME/.ssh/id_ed25519:/key:ro
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 2s
      timeout: 5s
      retries: 5
      start_period: 1s
  httpbin:
    image: kennethreitz/httpbin
    depends_on:
      tunmgr:
        condition: service_healthy
    # labels: # or provide tunnel names and ports explicitly
    #   tunmgr.names: httpbin # Comma separated list of names. Can be an empty
    #   tunmgr.ports: 80:80 $ Comma separated list of port maps. (remote:local)
    command: gunicorn -b 0.0.0.0:80 httpbin:app -k gevent --access-logfile -
```

With that docker compose file `httpbin` will be exposed as a public service on
tuns.

# How do I keep a tunnel open?

If you don't want to use `tunmgr` then we highly recommend using a tool like
[autossh](https://github.com/Autossh/autossh) to automatically restart a SSH
tunnel if it exits.

```bash
autossh -M 0 -R dev:80:localhost:8000 tuns.sh
```

# Bash script

Here's a helper script that should make it easier to create tunnels.

```bash
tunnel() {
    TUNNEL_TYPE=""
    TUNNEL_ENDPOINT=""
    TUNNEL_ARGS=""

    case $1 in
        http|https)
            TUNNEL_TYPE="-R"
            TUNNEL_ENDPOINT="$([[ $1 == "http" ]] && echo "80" || echo "443"):"

            if [ -z $2 ]; then
                echo "Tunnel provided incorrect port. Usage: tunnel $1 <port>"
                return
            fi

            if [ ! -z $3 ]; then
                TUNNEL_ENDPOINT="$3:${TUNNEL_ENDPOINT}"
            fi

            LOCAL_PORT=$2
            if [[ $LOCAL_PORT != *":"* ]]; then
                LOCAL_PORT="localhost:$2"
            fi

            TUNNEL_ENDPOINT+="$LOCAL_PORT"
            echo "Starting ${1^^} tunnel to $LOCAL_PORT"
            ;;
        tcp)
            TUNNEL_TYPE="-R"
            TUNNEL_ENDPOINT="${3:-0}:"

            if [ -z $2 ]; then
                echo "Tunnel provided incorrect port. Usage: tunnel $1 <port>"
                return
            fi

            LOCAL_PORT=$2
            if [[ $LOCAL_PORT != *":"* ]]; then
                LOCAL_PORT="localhost:$2"
            fi

            TUNNEL_ENDPOINT+="$LOCAL_PORT"
            echo "Starting ${1^^} tunnel to $LOCAL_PORT"
            ;;
        *)
            echo "unknown tunnel"
            return
            ;;
    esac

    ssh $TUNNEL_TYPE $TUNNEL_ENDPOINT tuns.sh $TUNNEL_ARGS
}
```

Example usage:

```bash
./tunnel.sh https 3000
./tunnel.sh tcp 1337
```

# UDP Tunneling

## Easy (`-o Tunnel=point-to-point`)

Using `tuns`, you have the ability to tunnel UDP traffic without any external
binary, meaning all using SSH. This makes use of the SSH tunneling functionality
and a `tun` interface. To get started, you need to follow a few steps:

1. Start some UDP service that you want to forward. For example, a simple socat
   echo server:

   ```bash
   socat -v PIPE udp-recvfrom:5553,fork
   ```

2. SSH into tuns requesting a `tun` interface with the information of where the
   service is running. This needs to be done as root. Replace
   `local-ip-of-machines-main-interface` with the ip address of the main
   interface for proper routing.

   ```bash
   sudo ssh -w 0:0 tuns.sh \
     udp-forward=10000:local-ip-of-machines-main-interface:5553
   ```

3. Bring the tunnel interface up and assign an ip that is link local (also as
   root):

   ```bash
   ip link set tun0 up; ip r a 10.1.0.1 dev tun0
   ```

4. Start a udp client to tuns.sh:10000. Here's one with netcat:

   ```bash
   nc -u tuns.sh 10000
   ```

## Hard (`-o Tunnel=ethernet`)

You can also use an ethernet tunnel for UDP forwarding. This makes a `tap`
interface. This is considered "hard mode" since you'll also need to handle ARP.
We don't process ARP packets, but we expect you to be an expert to be able to
make this work! The `SRC` interface `MAC` is `00:00:00:00:00:01`, while the
`DST` interface `MAC` is `00:00:00:00:00:02`

1. Start some UDP service that you want to forward. For example, a simple socat
   echo server:

   ```bash
   socat -v PIPE udp-recvfrom:5553,fork
   ```

2. SSH into tuns requesting a `tap` interface with the information of where the
   service is running. This needs to be done as root. Replace
   `local-ip-of-machines-main-interface` with the ip address of the main
   interface for proper routing.

   ```bash
   sudo ssh -o Tunnel=ethernet -w 0:0 tuns.sh \
     udp-forward=10000:local-ip-of-machines-main-interface:5553
   ```

3. Bring the tunnel interface up and assign an ip that is link local (also as
   root). You need to set the ARP entry and interface `MAC` as well:

   ```bash
   ip link set dev tap0 address 00:00:00:00:00:02
   ip link set tap0 up
   ip r a 10.1.0.1 dev tap0
   ip neigh add 10.1.0.1 lladdr 00:00:00:00:00:01 dev tap0 nud permanent
   ```

4. Start a udp client to tuns.sh:10000. Here's one with netcat:

   ```bash
   nc -u tuns.sh 10000
   ```

<hr />
<div class="flex flex-col items-center justify-center">
  <p>Create an account using only your SSH key.</p>
  <a href="/getting-started" class="btn-link">Get Started</a>
</div>
