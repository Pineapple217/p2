---
title: "Homelab access anywhere"
date: 2023-09-01T13:20:43+02:00
draft: false
hideToc: false
tags:
  - Homelab
series:
---

## Intro

To truly harness the potential of your Proxmox powered Homelab, there's a need for seamless and secure access from anywhere in the world. This is achieved by using a mesh VPN.

We'll go into setting up a Homelab mesh VPN using [Headscale](https://github.com/juanfont/headscale) and [Tailscale](https://tailscale.com/). Headscale is simpley a selfhosted Tailscale control server. We'll not go into how meshnetworks work.

In the end you (and only you) will have access to al your selfhosted services all over the world. We could even link multiple servers together securely.

If you want to go fully selfhosted by using Headscale you wil need an VPS (virtual private server) or an other kind of server with a puplic IP address.

## Setting up the Headscale control server

This server manages everything and needs to have a public IP address. Typically, this will be a VPS. This server does not interact with the network and is not part of it. This can be easily achieved using Docker.

```yml
# docker-compose.yml
version: "3.5"

services:
  headscale:
    container_name: headscale
    image: headscale/headscale
    restart: unless-stopped
    volumes:
      - ./config:/etc/headscale
      - ./data:/var/lib/headscale
    entrypoint: headscale serve
    networks:
      proxy-network:

  headscale-ui:
    container_name: headscale-ui
    image: ghcr.io/gurucomputing/headscale-ui:latest
    restart: unless-stopped
    networks:
      proxy-network:

networks:
  proxy-network:
    external: true
    name: proxy-network
```

In addition to the compose file, a config file is needed. You can find the default configs (config.yaml) on the Headscale GitHub. Some modifications are required in the config.yaml file, but nothing else needs to be changed in the default configs. Here are the relevant modifications:

```yaml
server_url: https://headscale.my-domain.be:443
listen_addr: 0.0.0.0:8080
metrics_listen_addr: 0.0.0.0:8080
```

The file structure should look like this. The 'data' directory will be created automatically when the containers are started, so you can ignore it for now.

```tree
.
├── config
│   └── config.yaml
├── data
│   ├── db.sqlite
│   ├── noise_private.key
│   └── private.key
└── docker-compose.yml
```

Finally, you need to set up the reverse proxy. Here, we use Caddy. In the docker-compose.yml file, you can see that both containers are part of the 'proxy-network.' The Caddy container is also part of this network. You should add the following to the Caddyfile:

```caddy
headscale.my-domain.be {
    basicauth /web* {
        user PASSWORDHASH_HERE
    }
    reverse_proxy /web* https://headscale-ui {
        transport http {
            tls_insecure_skip_verify
        }
    }
    reverse_proxy * http://headscale:8080
}
```

Basicauth is not necessary, but it adds an extra layer of security. Without it, people could potentially use your server as a frontend to connect to their backend. To avoid this, it's recommended to use Basicauth.

Lastly, for the root server, the web UI requires an API key. You can generate it with the following command:

```shell
docker exec headscale headscale apikeys create
```

Now, go to 'headscale.my-domain.be/web.' Under 'Settings,' you'll find where to enter the API key. Since the UI and the server are under the same domain name, you can leave the 'Headscale URL' field empty.

The server is now ready for use, and the next step is to add clients.

## Clients

### Proxmox

We'll use an LXC container because this service is very lightweight. You can create an Unprivileged container (check the Unprivileged container checkbox). I used Ubuntu 22 for this example, but you can use a different distribution if you prefer.

Configure the container with the following specifications:

- 4GB disk (can be smaller)
- 1 CPU
- 512 MB RAM
- Static IP (preferred but not required)

You can start the container immediately. After starting, it's a good idea to update it with the following commands (for Ubuntu):

```sh
apt update
apt upgrade -y
```

Once the update is complete, install the Tailscale client. You can find instructions on the [Tailscale website](https://tailscale.com/download/linux). At the time of writing, you can use a one-liner to install it. For Ubuntu, you need to install 'curl' first with:

```sh
apt install curl -y
```

Then, run the following command to install Tailscale:

```sh
curl -fsSL https://tailscale.com/install.sh | sh
```

After this is done, we need to make some additional configurations to allow Tailscale access to other devices on your network.

Edit the '/etc/sysctl.conf' file and uncomment the following lines:

```conf
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
```

Even if you don't use IPv6, it's recommended to enable both. After making these changes, shut down the container.

Once the container is stopped, you need to modify the container configuration, which is located on the host machine. In your case, it's located at '/etc/pve/lxc/120.conf', but the number may vary depending on the container ID. Add the following lines at the end of the file:

```conf
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

Now, start the container again. You can start the Tailscale client with the following command:

```sh
tailscale up --advertise-routes=10.0.0.0/24 --login-server https://headscale.my-domain.be
```

The 'advertise-routes' flag depends on your network configuration. Additionally, you can add the '--advertise-exit-node' flag if you want to use this network as a traditional VPN, allowing other devices to use it as an exit node.

After running this command, you'll receive a link. Open this link and copy the entire public key. You can then add this key via 'headscale.my-domain.be/web.'

### Android

Download the official Tailscale app. When you open it, repeatedly tap on the three dots in the upper-right corner until you see the 'Change server' option. Enter 'headscale.my-domain.be' here. Then, add the key through the web interface.
