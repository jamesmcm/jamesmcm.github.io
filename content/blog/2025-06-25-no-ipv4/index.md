+++
title = "Using the Internet without IPv4 connectivity"
date = 2025-06-25
[taxonomies]
categories = ["Linux"]
+++

A few days ago my ISP broke the IPv4 connectivity from my router after a
power cut. Fortunately IPv6 connectivity still worked fine, but only a
small fraction of websites were accessible.

In this post
I'll cover how Linux, WireGuard, and Hetzner came to the rescue - keeping the whole internet usable
with only an IPv6 connection.

<!-- more -->

Opinions expressed are solely my own and do not express the views or opinions of my employer.

## Background

One morning I woke up with no power, and noticed the breakers had triggered. After
retriggering them, everything went back to normal - except I couldn't
connect to Github and many other websites.

During the process of contacting my ISP, I realised the issue was only
with IPv4 servers. I could connect fine to IPv6 servers - and this
explained why Google and Meta worked fine, but a lot of websites
didn't. `ping -6` and `traceroute` on both my machine and the router's
diagnostic page confirmed this immediately. It seems this was an issue
with the [Carrier Grade NAT (CG-NAT)](https://en.wikipedia.org/wiki/Carrier-grade_NAT),
and this is why only IPv4 was affected.

Unfortunately, the ISP said they might need to send someone and it would
take several days, after the weekend too. Meanwhile I needed to be able
to access work and my wife needed to finish her thesis, so just leaving
it broken wasn't an option.

Fortunately, I remembered I had a Hetzner VPS server with both static
IPv4 and IPv6 addresses. And luckily, the Hetzner website supports IPv6
so I was able to access the console there to set things up.

But first we need to understand what Network Address Translation (NAT) is.
https://tailscale.com/blog/how-nat-traversal-works

## The WireGuard tunnel

The plan was simple: set up WireGuard on the VPS, and then use the IPv6
address as the endpoint in the client-side on my machine. Once the
tunnel is established, IPv4 traffic should then work as normal (albeit with higher latency
via the VPS) - effectively running our own
[Dual-Stack Lite](https://en.wikipedia.org/wiki/IPv6_transition_mechanism#Dual-Stack_Lite_\(DS-Lite\))
(DS Lite - but not the Nintendo console!).

I had previously used
[vps2arch](https://github.com/felixonmars/vps2arch) on my server, to
install Arch Linux
there, which works very well - and I was glad I did it, since it now
meant I had a familiar environment. I used the latest Debian image on Hetzner as the base.
Note you can probably also install Arch Linux manually afterwards by using Hetzner's ISO image mounting (they host
Arch Linux's ISO too, you don't need to set up a custom one).

At first I had some difficulties getting IPv6 traffic to work in the
tunnel (ironically, given the original issue), but eventually got it
working. Here is my full config for reference (adapted from
[the example on the ArchWiki](https://wiki.archlinux.org/title/WireGuard#Specific_use-case:_VPN_server),
but adding IPv6 traffic):

### Server-side

The server-side config:

```ini
# This is the server config
# Place in /etc/wireguard/wg0.conf
# Install wireguard-tools
# Then start with:
# sudo wg-quick up wg0
# You can also set up a service with systemd
# systemctl enable wg-quick@wg0.service
# systemctl start wg-quick@wg0.service

# Generate WireGuard keypairs with:
# wg genkey | (umask 0077 && tee peer_A.key) | wg pubkey > peer_A.pub
# Do this once for the server pair, and once for each client pair

[Interface]
Address = 10.200.200.1/24, fd42:42:42::1/64
ListenPort = 51820
PrivateKey = serverprivatekey # CHANGEME: Set server private key here

# Note here we assume the network device interface is eth0 - remember to check this!
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostUp = ip6tables -A FORWARD -i %i -j ACCEPT; ip6tables -A FORWARD -o %i -j ACCEPT; ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostUp = echo 1 > /proc/sys/net/ipv6/conf/all/forwarding
PostUp = echo 1 > /proc/sys/net/ipv4/conf/all/forwarding
PostUp = echo 1 > /proc/sys/net/ipv4/ip_forward

PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
PostDown = ip6tables -D FORWARD -i %i -j ACCEPT; ip6tables -D FORWARD -o %i -j ACCEPT; ip6tables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
# foo
PublicKey = clientpublickey # CHANGEME: Set client public key here
AllowedIPs = 10.200.200.2/32, fd42:42:42::2/128

[Peer]
# bar
PublicKey = client2publickey # CHANGEME: Set client2 public key here
AllowedIPs = 10.200.200.3/32, fd42:42:42::3/128
```

Note that unlike many .ini-style configuration files, wg-quick allows you
to specify PostUp and PostDown multiple times, and it will execute each command
in order.

#### IPv6 NAT via MASQUERADE

Also note here I am also using NAT with IPv6 (not only IPv4, where it is
necessary), this was just because I
had some problems at first getting direct routing to work, but in theory if you
have a block of address for your VPS (e.g. Hetzner gives you a /64 block
for IPv6) you can avoid NAT and make the Wireguard peers directly
addressable via IPv6.

Simply change the Unique Local Addresses (ULAs)
of the peers and interface to the public IPv6 addresses directly and
then remove the ip6tables MASQUERADE rule. Now each of the peers will be
directly addressable from Internet with their allocated IPv6 address.

If you want to forward
several devices with their own services this would definitely be the way
to go (but you also need to be sure the firewall rules on the VPS
correctly handle incoming traffic).

#### IPv4 SNAT

Finally note you could also use SNAT instead of MASQUERADE if you have a
static IPv4 address on the VPS and are certain it won't
change. This will be slightly more efficient, but didn't seem worth the
effort vs. having a portable config.

### Client-side

The client-side config:

```ini
# This is the client config, run on the client machine with:
# sudo wg-quick up ./foo.conf
# from wireguard-tools
[Interface]
Address = 10.200.200.2/32, fd42:42:42::2/128
PrivateKey = clientprivatekey # CHANGEME: Set client private key here
# Google DNS
DNS = 8.8.8.8
DNS = 2001:4860:4860::8888
MTU = 1280 # This was not in the initial config - see later in the post

[Peer]
PublicKey = serverpublickey # CHANGEME: Set server public key here
# Note the square brackets needed for IPv6
Endpoint = [serveripv6]:51820 # CHANGEME: Change serveripv6 here! If IPv4 do not need square brackets
AllowedIPs = 0.0.0.0/0, ::/0
```

Then with it running on both sides, everything worked smoothly. I could
even directly SSH into the tunnel-local IPv4 and IPv6 addresses to access the server
now.

This solved the issue of regular browsing, and it was also trivial to
install the WireGuard client on Linux for my wife.

However, I still couldn't connect to my work VPN like this since it
would interfere with the WireGuard connection.

## Network namespaces

As the creator of [vopono](https://github.com/jamesmcm/vopono), my
plan was to run the work VPN and any
necessary applications in a network namespace. The trick is to set the
MASQUERADE rule so that it forwards the traffic to the WireGuard
interface (foo above), instead of directly to the actual network
interface (enpXsY).

This way the traffic inside the network namespace is oblivious to the
WireGuard tunnel nftables rules (from wg-quick) outside on the host,
but the traffic will be routed through the WireGuard tunnel.

It's also worth noting that despite wg-quick preferring to use nftables over
iptables when available, it manages to avoid conflicts with
Docker's standard iptables rules.


![Architecture Overview](./netns_diag.png)


We can do this with vopono by specifying the running WireGuard interface
with the `-i` argument. All together it looks like this:

```sh
$ vopono -v exec --create-netns-only --provider None --protocol None -i foo bash
$ sudo ip netns exec vo_none_none bash
$ (inside netns) ./vpn.sh  # Script to run the work VPN
```

Note that `/etc/netns/vo_none_none/` will be mounted to `/etc` in a
mount namespace by `ip netns exec`. This means we can put specific DNS
servers in that `resolv.conf`. Note you can do the same to edit
`gai.conf` if you want to give precedence to IPv4 DNS resolution in the network
namespace for example.

I used the latter when I was still struggling to get IPv6 traffic to
work through the tunnel, here is the `gai.conf` for reference (adapted
from this
[AskUbuntu answer](https://askubuntu.com/questions/32298/prefer-a-ipv4-dns-lookups-before-aaaaipv6-lookups)):

```conf
precedence ::ffff:0:0/96 100
#    For sites which use site-local IPv4 addresses behind NAT there is
#    the problem that even if IPv4 addresses are preferred they do not
#    have the same scope and are therefore not sorted first.  To change
#    this use only these rules:
#
scopev4 ::ffff:169.254.0.0/112  2
scopev4 ::ffff:127.0.0.0/104    2
scopev4 ::ffff:0.0.0.0/96       14
```

So after connecting to the VPN as above, we can put the internal DNS
servers in `/etc/netns/vo_none_none/resolv.conf` so everything works for the next applications launched in
the network namespace. It's a bit awkward that we have to do this after
connecting (since before we can't access these servers), but since the
mount namespace is specific to each `ip netns exec` invocation, the
script can't do it for us (unless we run everything inside that same
bash session, and don't use `ip netns exec` again).

We can then run applications as our normal user in the network namespace
(now going through the work VPN, with the `vpn.sh` script running in
another session):

```sh
$ vopono -v exec -i foo --provider None --protocol None google-chrome-stable
```
But there's still one remaining issue to be able to run everything
needed via the VPN: Docker.

## Docker

Naively running Docker just like other applications above will not
work - the Docker socket was
created outside the network namespace (when enabled with systemd), and
so we won't have any internal connectivity.

However, stopping Docker externally and trying to simply run the Docker
daemon and create the socket within the network namespace also won't
work - this is due to `ip netns exec` creating a mount namespace and
remounting `/sys` - so our host's `/sys/fs/cgroup` won't be visible.

This will give an error like:

```
Error: OCI runtime error: runc: runc create failed: no cgroup mount found in mountinfo
```

(note I think that error was actually from testing with podman-docker)

We can hack around this with the following commands:

```sh
$ (on host) sudo systemctl stop docker && sudo systemctl stop docker.socket
$ (on host) sudo -E unshare -m sh -c 'mount --bind /sys /sys; exec ip netns exec vo_none_none sudo --user archie --preserve-env bash'
$ (in netns) sudo umount /sys
$ (in netns) sudo dockerd --host=unix:///var/run/docker-netns.sock --data-root=/var/lib/docker-netns --default-runtime=runc
$ (in netns) DOCKER_OPTS="--dns=YOURDNSHERE" DOCKER_HOST=unix:///var/run/docker-netns.sock sudo --user archie --preserve-env docker ... # your docker command here
```

This was adapted from [this Unix StackExchange post](https://unix.stackexchange.com/questions/686155/how-can-i-use-a-bind-mount-in-a-network-namespace).
The trick is to use `unshare` to force `/sys` to be a bind mount in the mount namespace
created by `ip netns exec`, and then unmount the internal `/sys` mount created by 
`ip netns exec` - so now `/sys` inside the mount namespace is a bind
mount of the host's `/sys`.

This will only affect this invocation
of `ip netns exec`. But we can then start `dockerd` and our Docker
command in the same session inside the network namespace, so the mount
namespace is the same for both of them.

Note you can also set the Docker DNS settings in
`/etc/netns/vo_none_none/docker/daemon.json`.

Also note this worked for my needs (including one container that uses a
Docker network to connect to a helper container), but it may not work for more
complex Docker set-ups that need bridges,
etc. (although I am sure it is technically possible).

## WireGuard MTU issues

After rebooting my machine I had some issues with the WireGuard
connection where only a few pages would load, and some wouldn't at all -
e.g. Github, meanwhile `ping` and `ping -6` worked perfectly.

This was very hard to debug - as even `wg show` showed the WireGuard
connection functioning fine. In the end I found it by doing pings of
different sizes:

```sh
$ ping6 -s 1400 fd42:42:42::1
$ ping6 -s 1200 fd42:42:42::1
$ ping6 -s 800 fd42:42:42::1
```

In this case the first one failed, but the others worked. This revealed 
that the connection issues were due to the WireGuard MTU
setting - larger packets were being dropped. Setting a lower MTU fixed the issues immediately (and is
reflected in the config above).

## Conclusion

In summary, we've covered:

- Creating a WireGuard VPN server on a VPS with IPv4 and IPv6 traffic.
- Using a network namespace to run another VPN over this WireGuard
  interface.
- Using `unshare` tricks to run Docker inside that network namespace.

Internet connectivity issues are always a risk when working remotely,
fortunately in this case Linux was able to save the day and save me from
being entirely at the mercy of whenever the ISP fixes the
configuration.

I hope this is useful to others (and perhaps even myself again in the
future!). It has really demonstrated the benefits of Linux's "fix it
yourself" approach. While Macs are tempting with their great M4
processors, I'd have no idea how to manage all of the above on MacOS
(and [my last MacOS experience wasn't great!](@content/blog/2020-09-20-catalina-cachedelete.md)).

I highly recommend Hetzner for a VPS, they have great prices and
fully support running WireGuard tunnels and pretty much all legitimate
usage (just no port scanning, traffic spoofing or cryptocurrency mining).
You never know when you might need it. (Note another useful option here might
be a VPN like AirVPN, ProtonVPN or AzireVPN that supports port forwarding so you
can forward ports from your own home servers without relying on your
ISP).

It's also made me consider getting an OpenWRT router for the same
reason. I used to think that managing your own router is unnecessary
extra work (ironically how a lot of people probably think about
GNU/Linux), but being able to debug more on the router side would be
great for problems like this, or even just running Wireguard on it
directly for this work-around without having to configure it on every
device separately.

