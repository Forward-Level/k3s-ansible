# K3S for Forward Level with Ansible

## INTRODUCTION

This repository and playbook represents the configuration for running forwardlevel.com in a high-availability (HA) K3S kubernetes deployment.

## REQUIREMENTS

On the controlling device (eg, my local Mac):

-   Ansible
    -   `brew install ansible`
    -   On Ubuntu, `apt-get install pipx -y && pipx install --include-deps ansible`

On the deployment servers:

-   TBD

Optionally

-   Tailscale

## INSTALLATION STEPS

This process utilizes the `k3s-io/k3s-ansible` repository at it's core [https://github.com/k3s-io/k3s-ansible](https://github.com/k3s-io/k3s-ansible) as a git submodule.

When initially building the repo:
`git submodule add https://github.com/k3s-io/k3s-ansible`

### 1. Cloning the Ansible deployment project (with git submodules)

When you clone such a project, by default you get the directories that contain submodules, but none of the files within them yet.

Alternatively, run:

`git clone --recurse-submodules git@github.com:Forward-Level/k3s-ansible.git`

It will automatically initialize and update each submodule in the repository, including nested submodules if any of the submodules in the repository have submodules themselves.

If you want to do it the hard way, `git clone xyz` …downloads as normal but, while the submodule directory is there, it is empty. You must run the following two commands:

    1. `git submodule init` to initialize your local configuration file, and
    2. `git submodule update` to fetch all the data from that project and check out the appropriate commit listed in your superproject.

### 2. Ansible

**Installing the Ansible dependencies** first:

**IMPORTANT!!!** You will need to install collections that the Ansible playbook uses by running

    `ansible-galaxy collection install -r ./collections/requirements.yml`

Additionally, the `netaddr` package must be available to Ansible. If you have installed Ansible via apt, this is already taken care of. If you have installed Ansible via pip, make sure to install netaddr into the respective virtual environment. In terms of Homebrew on the Mac, I installed using `brew install netaddr`.

Edit the `inventory/my-cluster/hosts.ini` file with the IPs of our control servers and our worker nodes:

    ```text
    [master]
    192.168.30.38
    192.168.30.39
    192.168.30.40

    [node]
    192.168.30.41
    192.168.30.42

    [k3s_cluster:children]
    master
    node
    ```

### 2a Preparing test servers:

#Create users on all servers:
sudo groupadd -g 1000 osho && sudo useradd -u 1000 -g 1000 -d /home/osho -s /bin/bash osho && sudo passwd osho
sudo groupadd -g 1000 osho && sudo useradd -u 1000 -g 1000 -d /home/osho -s /bin/bash osho && sudo adduser osho sudo

#install tailscale
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
tailscale ip

100.75.31.27 100.112.208.64 100.68.11.93 100.74.96.126 100.91.134.52

100.75.31.27
100.112.208.64
100.68.11.93
100.74.96.126
100.91.134.52

#create folders
for ip in 100.75.31.27 100.112.208.64 100.68.11.93 100.74.96.126 100.91.134.52; do ssh root@$ip 'mkdir /home/osho && sudo chown osho /home/osho && mkdir /home/osho/.ssh && sudo chown osho /home/osho/.ssh'; done

#copy keys
for ip in 100.75.31.27 100.112.208.64 100.68.11.93 100.74.96.126 100.91.134.52; do ssh-copy-id -i /Users/andy/.ssh/id_rsa.pub osho@$ip; done

#confirm users and keys
for ip in 100.75.31.27 100.112.208.64 100.68.11.93 100.74.96.126 100.91.134.52; do ssh osho@$ip 'whoami && hostname'; done

### 3. Adventures in load balancing: configuration of nginx as a HTTP/HTTPS load balancer

Installation:

`sudo apt-get update && sudo apt-get install nginx`

Once installed change the directory into the nginx main configuration folder.

`cd /etc/nginx/`

Ubuntu and Debian follow a rule for storing virtual host files in `/etc/nginx/sites-available/`, which are enabled through symbolic links to `/etc/nginx/sites-enabled/`.

Use the command below to enable any new virtual host files.

In the context of nginx, the `sites-available` directory is typically used to store all of your server block configurations, even ones that aren't currently enabled. The `sites-enabled` directory is used to store links to the configurations in `sites-available` that you want to enable. nginx will only use the configurations linked in the `sites-enabled` directory.

So, this command is enabling the `vhost` server block configuration by creating a symbolic link to it in the `sites-enabled` directory.

`sudo ln -s /etc/nginx/sites-available/vhost /etc/nginx/sites-enabled/vhost`

Check that you find at least the default configuration and then restart nginx.

`sudo systemctl restart nginx`

Test that the server replies to HTTP requests. Open the load balancer server’s public IP address in your web browser. When you see the default welcoming page for nginx the installation was successful.
nginx default welcome page.

If you have trouble loading the page, check that a firewall is not blocking your connection. For example on CentOS 7 the default firewall rules do not allow HTTP traffic, enable it with the commands below.

`sudo firewall-cmd --add-service=http --permanent`
`sudo firewall-cmd --reload`

With **[HTTP Load Balancing](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/)**: one of the things needed of the nginx load balancer is some sort of approach to prioritize the fallback workers.

Load balancing with nginx uses a round-robin algorithm by default if no other method is defined, like in the first example above. With a round-robin scheme, each server is selected in turns according to the order you set them in the load-balancer.conf file. This balances the number of requests equally for short operations.

Least connections-based load balancing is another straightforward method. As the name suggests, this method directs the requests to the server with the least active connections at that time. It works more fairly than round-robin would with applications where requests might sometimes take longer to complete.

To enable the least connections balancing method, add the parameter `least_conn` to your `upstream` section as shown in the example below.

Round-robin and least connections balancing schemes are fair and have their uses. However, they cannot provide session persistence. If your web application requires that the users are subsequently directed to the same back-end server as during their previous connection, use the IP hashing method instead. IP hashing uses the visitor’s IP address as a key to determining which host should be selected to service the request. This allows the visitors to be each time directed to the same server, granted that the server is available and the visitor’s IP address hasn’t changed.

To use this method, add the `ip_hash` parameter to your `upstream` segment like in the example underneath.

In our case, we'll use `ip_hash` in our configuration.

**Health checks** are great.

In order to know which servers are available, nginx’s implementations of reverse proxy include passive server health checks. If a server fails to respond to a request or replies with an error, nginx will note the server has failed. It will try to avoid forwarding connections to that server for a time.

The number of consecutive unsuccessful connection attempts within a certain time period can be defined in the load balancer configuration file. Set a parameter `max_fails` on the server lines. By default, when no `max_fails` is specified, this value is set to `1`. Optionally setting the `max_fails` to `0` will disable health checks to that server.

If `max_fails` is set to a value greater than `1` the subsequent fails must happen within a specific time frame for the fails to count. This time frame is specified by a parameter `fail_timeout`, which also defines how long the server should be considered failed. By default, the `fail_timeout` is set to 10 seconds.

After a server is marked failed and the time set by `fail_timeout` has passed, nginx will begin to gracefully probe the server with client requests. If the probes return successful, the server is again marked live and included in the load balancing as normal.

    ```text
    upstream backend {
        server 10.1.0.101 weight=5;
        server 10.1.0.102 max_fails=3 fail_timeout=30s;
        server 10.1.0.103;
    }
    ```

Use the health checks. They allow you to adapt your server back-end to the current demand by powering up or down hosts as required. When you start up additional servers during high traffic, it can easily increase your application performance when new resources become automatically available to your load balancer.

**Our standard configuration** is as follows:

    ```text
    # Define which servers to include in the load balancing scheme
    # It's best to use the servers' private IPs for better performance and security

    http {
        upstream backend {
            least_conn;         # session balancing -- other option is random
            ip_hash;            # session stickiness
            # LINODE_WORKER1
            server 45.79.103.31 weight=100 max_fails=3 fail_timeout=10s;
            # LINODE_WORKER2
            server 45.79.103.165 weight=100 max_fails=3 fail_timeout=10s;
            # FALLBACK_WORKER1
            server <FALLBACK_WORKER1_IP> weight=10 max_fails=3 fail_timeout=30s;
            # FALLBACK_WORKER2
            server <FALLBACK_WORKER2_IP> weight=5 max_fails=3 fail_timeout=30s;
            # FALLBACK_WORKER3
            server <FALLBACK_WORKER3_IP> weight=1 max_fails=5 fail_timeout=60s;
        }

        # For server weighting, in the configuration shown above
        # the primary servers are selected three times as often as
        # the fallbacks. The first fallback, `orbital` and the secondary
        # fallback servers (`satellite` and `andys-imac`) are given
        # twice the number of requests compared to `andys-imac`.

        # `LINODE_WORKER1` and `LINODE_WORKER2` have weights of 4;
        # the other two servers have the default weight (1), but
        # FALLBACK_WORKER1, FALLBACK_WORKER2, and FALLBACK_WORKER3 are
        # marked as backup servers and do not receive requests unless both
        # of the other servers are unavailable. With this configuration of
        # weights, out of every 6 requests, 5 are sent to backend1.example.com and 1 to backend2.example.com.

        # It's generally a good idea to have health checks on all servers.
        # Health checks help ensure that traffic isn't being sent to servers
        # that are down or not responding properly. If FALLBACK_WORKER3 is
        # intended to be a last-resort server, it might make sense to have a
        # different health check configuration for it, but it should still
        # have some form of health check.

        # In this configuration, FALLBACK_WORKER3 will be considered
        # unavailable after 5 failed attempts to communicate with it,
        # and it will be considered unavailable for 60 seconds.
        # This gives FALLBACK_WORKER3 more leeway than the other
        # servers before it's considered unavailable.

        # This server accepts all traffic to port 80 and passes it to the upstream
        # Notice that the upstream name and the proxy_pass need to match

        server {
            listen 80;

            location / {
                proxy_pass http://backend;
            }
        }
    }
    ```

Next, disable the default server configuration you earlier tested was working after the installation. Again depending on your OS, this part differs slightly.

On Debian and Ubuntu systems you’ll need to remove the default symbolic link from the sites-enabled folder.

`sudo rm /etc/nginx/sites-enabled/default`

Then use the following to restart nginx.

`sudo systemctl restart nginx`

Much of the above configuration came from [https://upcloud.com/resources/tutorials/configure-load-balancing-nginx](https://upcloud.com/resources/tutorials/configure-load-balancing-nginx)

### 3. Installation of k3s

Prerequisites:

    - `ansible` successfully installed.

    Ansible is an IT automation platform that allows you to utilize “playbooks” to manage the state of remote machines. It’s commonly used for managing configurations, deployments, and general automation across fleets of servers.

    Essentially, Ansible allows you to configure tasks that tell it what the system’s desired state should be; then Ansible will leverage modules that tell it how to shift the system toward that desired state.

    The only special software required is Ansible, running on one central device that manipulates the remote targets. If you wish to learn more about Ansible, the [official documentation](https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html) is quite extensive.

    NOTE: From the Rancher documentation, [Deploying k3s with Ansible](https://www.suse.com/c/rancher_blog/deploying-k3s-with-ansible/), regarding the Ubuntu installation phase:
    > The 'SSH Setup' screen is also important. The Ubuntu Server installation process lets you import SSH public keys from a GitHub profile, allowing you to connect via SSH to your newly created VM with your existing SSH key. To take advantage of this, [make sure you add an SSH key to GitHub](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account) before completing this step. This is highly recommended, as although Ansible can connect to your VMs via SSH using a password, doing so requires extra configuration. It is also generally good to use SSH keys rather than passwords for security reasons.

### 4. Installing Helm

### 5. Installing Rancher

Prerequisites:

    - `k3s` successfully installed

    - `kubectl` - Kubernetes command-line tool. Installed with the k3s. [https://kubernetes.io/docs/reference/kubectl/](https://kubernetes.io/docs/reference/kubectl/)

    - `helm` - Package management for Kubernetes. [https://helm.sh/docs/intro/install/](https://helm.sh/docs/intro/install/)

    Helm now has an installer script that will automatically grab the latest version of Helm and install it locally.

    You can fetch that script, and then execute it locally. It's well documented so that you can read through it and understand what it is doing before you run it.

    ```sh
    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
    chmod 700 get_helm.sh
    ./get_helm.sh
    ```

Rancher installation guide:
[https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/infrastructure-setup/ha-k3s-kubernetes-cluster](https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/infrastructure-setup/ha-k3s-kubernetes-cluster)

For the .k3s.env file:

    ```text
    RANCHER_SQL_HOST=
    RANCHER_LB_HOST=
    RANCHER_URL=
    ```

## ENV VARIABLES

You will need to configure the following environment variables in a file `.k3s.env` stored in the local directory.

    ```text
    LINODE_HOSTNAME_CONTROL1=k3s-control00
    LINODE_HOSTNAME_CONTROL2=k3s-control01
    LINODE_HOSTNAME_CONTROL3=k3s-control02
    LINODE_HOSTNAME_WORKER1=k3s-worker00
    LINODE_HOSTNAME_WORKER2=k3s-worker01

    LINODE_CONTROL1=
    LINODE_CONTROL2=
    LINODE_CONTROL3=
    LINODE_WORKER1=
    LINODE_WORKER2=

    RANCHER_SQL_HOST=
    RANCHER_LB_HOST=
    RANCHER_URL=
    ```

If using Tailscale to manage the network, provide the following:

    ```text
    TAILSCALE_CONTROL1=
    TAILSCALE_CONTROL2=
    TAILSCALE_CONTROL3=
    TAILSCALE_WORKER1=
    TAILSCALE_WORKER2=
    ```

Finally, I have provisioned three fallback workers that should be the lowest priority on the stack.

     - `FALLBACK_WORKER1` is on my personal Linode server, `orbital`
     - `FALLBACK_WORKER2` is an Ubuntu container on my homelab Proxmox server.
     - `FALLBACK_WORKER3` is an Ubuntu-based Docker container on my desktop at the office

I've set these three fallbacks up in a named order that implies their expected availability and internet speeds. Linode's fiber backbone connection is going to be the most reliable, and their uptime guarantee is pretty solid. My home internet plan is 1000mbps and the machines are on 24/7, but have no guarantees. At the office, the wifi is slow(er) and truly anything could happen.
