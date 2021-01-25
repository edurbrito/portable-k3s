# Portable HA k3s

I have been thinking about running Kubernetes but didn't know where to get started. 
[Techno Tim](https://github.com/techno-tim/youtube-videos/tree/master/k3s-ha-install) introduced me to [k3s](https://rancher.com/docs/k3s/latest/en/): 

> A highly optimized distribution of Kubernetes that can run on ARM, IoT, Raspberry Pis, or even x86 hardware. It's small, lightweight, and gives all the features in Kubernetes!

Here is my attempt at setting it up, virtualized, and hosted on my PC.

## Prerequisites

### Architecture

![k3s High Availability](images/k3s-architecture-ha-server.png)

Taken from the [Rancher Docs](https://rancher.com/docs/k3s/latest/en/architecture/#high-availability-k3s-server-with-an-external-db).

#### Virtualization Tool

My personal choice goes to [VirtualBox](https://www.virtualbox.org/).

#### Some Ubuntu Server Machines

In this case, I created 3 [Ubuntu Server](https://ubuntu.com/download/server) virtual machines (2048 MB of RAM and 8GB of Disk):
* ubuntu-1 - Kubernetes Agent
* ubuntu-2 - Kubernetes Server
* ubuntu-3 - Kubernetes Server

For minimal High Availability, 1 agent and 2 servers are needed, but you can spin up as many as you need.

All of them also have [Docker](https://www.docker.com/) and [OpenSSH](https://www.openssh.com/) installed and enabled at startup.

As a note, you may wish to configure their network adapters as "Bridge" adapters and, for maintainability, setting their IP addresses as static, doing the following (regarding the local network you are connected to):
1. `sudo nano /etc/netplan/00-installer-config.yaml` or whatever file you may find inside `netplan` folder.

```yaml
network:
ethernets:
    enp0s3:
    dhcp4: no
    addresses: [192.168.1.XX/24] # XX could be {90 => ubuntu-1,91 => ubuntu-2, 92 => ubuntu-3}
version: 2
```

2. `sudo netplan apply`

#### Creating an SQL Database

Inside my `ubuntu-1` machine, I have created a [MySQL](https://www.mysql.com/) docker container, with the port mapping `0.0.0.0:3306 -> 3306/tcp` and restarting policy as **Always**. 
After starting it, a new user with the name `k3s` and with all the permissions is needed for the Kubernetes engine.

#### Creating an NGINX load balancer

Also inside my `ubuntu-1` machine, an [NGINX](https://www.nginx.com/) docker container was created with the following configurations:

First, created these 2 files, the **Dockerfile** for the NGINX image and the **nginx.conf**, holding the desired configuration.

* `Dockerfile`
```Dockerfile
FROM nginx
COPY nginx.conf /etc/nginx/nginx.conf
```
* `nginx.conf`
```nginx
events {}

stream {
    upstream k3s_servers { // Add here the IP addresses of your server machines. In my case, I did the following:
        server 192.168.1.XX:6443; // ubuntu-2
        server 192.168.1.XX:6443; // ubuntu-3
    }

    server {
        listen 6443;
        proxy_pass k3s_servers;
    }
}
```

Then, just ran these two commands:

1. `docker build -t k3s-nginx .` for building the image.
2. `docker run --restart always --name k3s-nginx -d k3s-nginx` for starting the container.

And checked the address `192.168.1.XX:6443` (ubuntu-1) to see if it was running.

## Installing k3s

#### Servers

Inside my `ubuntu-2` and `ubuntu-3` server machines, I ran:

1. `export K3S_DATASTORE_ENDPOINT='mysql://username:password@tcp(database_ip_or_hostname:port)/database'` with username as `k3s` and the password the one I defined when creating this user in the database. The database IP was, in my case, `192.168.1.XX:3306` (ubuntu-1) and the database name also `k3s`.

> If you plan on installing Rancher in this cluster, the current stable release (v2.5.5) does not support K3S v1.20. To prevent issues when installing rancher, be sure to include the following:
>
> 2. `export INSTALL_K3S_VERSION=v1.19.5+k3s2`

3. `curl -sfL https://get.k3s.io | sh -s - server --node-taint CriticalAddonsOnly=true:NoExecute --tls-san load_balancer_ip_or_hostname` with the load balancer IP address as `192.168.1.XX:6443` (ubuntu-1).
4. `sudo chmod 644 /etc/rancher/k3s/k3s.yaml` to enable it to be run without `sudo` anymore.
5. `k3s kubectl get nodes` to check if everything is on the right track. Here you should see a table with all the machines already configured, so you may run this command later, after configuring them all.
6. `sudo cat /var/lib/rancher/k3s/server/node-token` to get the token. Save it for the next task, when configuring the agents.

#### Agents

Inside my `ubuntu-1`, I ran:

> If you plan on installing Rancher in this cluster, the current stable release (v2.5.5) does not support K3S v1.20. To prevent issues when installing rancher, be sure to include the following:
> 
> 1. `export INSTALL_K3S_VERSION=v1.19.5+k3s2`

2. `curl -sfL https://get.k3s.io | K3S_URL=https://load_balancer_ip_or_hostname:6443 K3S_TOKEN=mynodetoken sh -` with the load balancer IP address (in my case, again, was `192.168.1.XX:6443` (ubuntu-1)) and the node token from the last server command.

## Some nice GUI

As you noticed, along the way, there were some commands that could be used to enable [Rancher](https://rancher.com/) support. As a personal taste and for simplicity sake, I have used [Portainer](https://www.portainer.io/) to manage all my Docker and Kubernetes related things. To enable Portainer:
1. Install it on your host machine. I managed to use it as a docker container itself [this way](https://documentation.portainer.io/v2.0/deploy/linux/?hsCtaTracking=a66b69bb-4970-46b7-bc31-cfc8022c7eb2%7C0d5be9a2-9dac-4ab1-9498-4b07566effd3#deploy-portainer-in-docker).
2. Create a username and password.
3. Go to `Endpoints -> Agent - Portainer Agent -> Kubernetes Via Node Port`.
4. Copy the suggested command: `curl -L https://downloads.portainer.io/portainer-agent-k8s-nodeport.yaml -o portainer-agent-k8s.yaml; kubectl apply -f portainer-agent-k8s.yaml`
5. Run it in one of your server nodes. I ran this in my `ubuntu-2` machine.
6. Give it a name.
7. Set the `Endpoint URL` as the IP address of the server where you've just run the previous command, `192.168.1.XX:30778` (in my case, was ubuntu-2 and the port was the default Portainer port).
8. Create the Endpoint.
9. To test it, go to `Applications`.
10. Add a new `Application`.
11. Set its image as `nginx:latest` or whatever you wish.
12. Set the other configurations as you want with special attention to the `Instance Count` - the number of replicas of this application running in this cluster - try it with different values.
13. `Publishing the Application -> Cluster`.
14. `Published Ports: container port = 80 -> node port = 30080` (or whatever you want)
15. Deploy it and wait for all the `Replicas` to start.
16. Go to the IP address of your agent, `192.168.1.XX:30080` (ubuntu-1), and check if is something running there.
