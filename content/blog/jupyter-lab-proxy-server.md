---
title: "Server hopping and jupyter lab"
description: "Networking for the masses"
dateString: Dec 2022
draft: false
tags: ["SSH", "NETWORKING", "SERVER", "DATA SCIENCE", "JUPYTER"]
weight: 105
cover:
    image: "/blog/jupyter-lab-proxy-server/server.jpg"
    # caption: "Photo by Florian Krumm on Unsplash"
---

# Server hopping and jupyter lab

[Jupyter lab](https://jupyter.org/) has made the life of Data Scientists and researchers alike easier. With easy-to-run code blocks, rich outputs and customizable kernels you can now dynamically explore and conquer your data.

Running your notebooks locally is straightforward, typing `jupyter lab` in your terminal with a valid [installation of the jupyter library](https://jupyterlab.readthedocs.io/en/stable/getting_started/installation.html) should open a new browser tab with a new session. From here on, all the magic happens, but I'm not here to explain the newest unheard-of machine learning algorithm.

Instead, I want to shed light into a topic that not even the deepest Stackoverflow searches could solve for me. How to launch your **jupyter lab** session when working in a high performance computing (HPC) cluster **with a proxy server**.

If you don't need to bother about proxy servers, I highly recommend this other article which covers the basics in more detail:

<https://towardsdatascience.com/how-to-connect-to-jupyterlab-remotely-9180b57c45bb#bb1c>

## Jupyter lab in a HPC cluster

Let's first start with the basics. Once you *understand* how to launch your sessions without a proxy server, the problem of adding a proxy server to the process should be easier.

HPC clusters are common in research centres and universities. In a cluster, several servers or nodes are connected via a fast local network. This enables access to parallel computing, high memory and GPU acceleration.

In order to access our notebook from a remote machine we will need to tweak some jupyter options. First of all, we don't want a new browser window to automatically open in the server, so we add `--no-browser` to our command. Second, we need to specify the port we are going to serve our lab to. A port can be specified with `--port=*port_to_use*`*.* HPC clusters can be crowded withother users using ports in the same node, so the choice of port should be properly informed if you don't want to keep seeing `Port NNNN already in use`.

Jupyter will serve the lab session to `localhost`. In order to access it remotely we need to do some port forwarding. This is just instructing your computer to communicate through specific ports with the server.

Putting all of this together, a typical `launch_notebook.sh`script should be similar to:

```bash
function random_unused_port {

    local port=$(shuf -i 2000-65000 -n 1)

    netstat -lat | grep $port > /dev/null

    if [[ $? == 1 ]] ; then

        export RANDOM_PORT=$port

    else

        random_unused_port

    fi

}

JUPYTER_PORT=$RANDOM_PORT

jupyter-lab --no-browser --port=${JUPYTER_PORT}
```

> From here on I refer to the port that we are serving our session to as `${JUPYTER_PORT}`but the environment variable is only saved in the machine running `random_unused_port{}` . Locally, you will have to introduce the port manually.

Then we type the following locally:

```bash
ssh -L localport:localhost:${JUPYTER_PORT} user@remote_host
```

We can finally access our session by going to `localhost:local_port`in your web browser.

## Server hopping

![](/blog/jupyter-lab-proxy-server/server-hopping.jpg)

Proxy servers are an extra layer of security that computing clusters can use to avoid potential malicious exploits or misuse. In this case, access to the cluster is only possible via the proxy server. We can’t simply do port forwarding directly into the machine that’s serving our lab session.

To make our lives easier we can set up aliases in our `.ssh/config`file to access the cluster and the proxy server, separately. This would be enough if we were to be running our lab session in the login node. However, login nodes are not meant to be handling any computation loads, so we need to serve our session in a computation node. In order to do that we just need to modify our original `launch_notebook.sh`to include both the commands for the job scheduler (SLURM in my case) and to forward the ports to the login node from the compute node.

```bash
$ cat .ssh/config

Host work-via-jump

    Hostname {cluster IP address}

    User {user}

    ProxyCommand ssh user@{proxy IP address} -W %h:%p

    ServeraliveInterval 60

Host jump-server

    Hostname {proxy IP address}

    User {user}

    ServeraliveInterval 60
```

The first step is to ssh to the cluster, `ssh work-via-jump`should be enough now that you have a `config`file. Once in the login node, we need to serve our session in a computation node. In order to do that we just need to modify our original `launch_notebook.sh`to include both the commands for the job scheduler (SLURM in my case) and to forward the ports to the login node from the compute node.

```bash
source $HOME/.bashrc

module load tools/jupyterlab

function random_unused_port {

    local port=$(shuf -i 2000-65000 -n 1)

    netstat -lat | grep $port > /dev/null

    if [[ $? == 1 ]] ; then

        export RANDOM_PORT=$port

    else

        random_unused_port

    fi

}

random_unused_port

JUPYTER_PORT=$RANDOM_PORT

ssh -R localhost:${JUPYTER_PORT}:localhost:${JUPYTER_PORT} {login_node IP or alias} -N &

jupyter-lab --no-browser --port=${JUPYTER_PORT}
```

The second step is to communicate our computer with the login node in the cluster — remember that the login node is communicating with the compute node already — and with the proxy server. Here, port forwarding to both of these is quite straightforward. We will need two terminal sessions to be running in parallel. The first one will forward ports to the proxy server whilst the second one will be running on the proxy server and forwarding ports to the login node.

```bash
$ ssh -f jump-server -L ${JUPYTER_PORT}: localhost:${JUPYTER_PORT} -N
```

In another terminal:
```bash
$ ssh -X jump-server

$ ssh -f user@{IP address} -L ${JUPYTER_PORT}:localhost:${JUPYTER_PORT} -N -K
```

Now you should be able to access the session in your browser by typing `localhost:${JUPYTER_PORT}`.

> I consciously skipped some of the ssh command details and flags since I just wanted to provide the solution _as is_. Let me know if you want me to go in more detail with these.
