---
title: "How to Setup SSH using Tailscale or ngrok"
date: 2020-05-29T18:05:06+05:30
tags: ["ssh", "ngrok", "tailscale", "networking"]
slug: "ssh using tailscale or ngrok"
---
<style>
.caption {
    font-size: 0.9em;
    margin: 0px 50px;
    text-align: center;
    margin-bottom: 20px;
}

</style>

![](/cables.jpeg)
SSH is a secure protocol used as the primary means of connecting to Linux servers remotely.
The remote server can be your computer. For instance, I use SSH to access my office computer.

Setting up SSH is easy, and there are multiple articles on how to set up SSH. This blog post is 
about how to SSH into a machine that does not have a static public IP; that is, the machine is 
inside a private network, for example, my laptop at home or my computer inside a WeWork building.

# Tailscale and Ngrok to the Rescue
[Tailscale](http://tailscale.io/) and [ngrok](https://ngrok.com/) are quite similar in what they do.
In simple terms, both the services will install a software on your computer, which will help in 
routing the traffic from your private network into the internet. ngrok allows you to expose specific 
ports to the internet, while tailscale creates a virtual private network for you. If your goal is to 
SSH into a machine inside a private network, both of them will do the job.

![](/ngrok.png)
<div>
<div class="caption">Visual representation of how ngrok works - from https://ngrok.com/product</div>
</div>

# SSH using Tailscale
Tailscale is the easiest way of exposing the SSH server. All you have to do is install tailscale by 
following the [instructions here](https://tailscale.com/download).
Once it is set up, you're ready to connect to the machine.

Run the following command on your host machine (the machine you wish to connect to)
```
ip addr show tailscale0
```
The output of the command above will show you the IP address tailscale is using. This IP address 
never changes. Now, to connect to this machine, run the following command on your client machine 
(the machine you're using to connect to the host machine)

```
ssh <username>@<tailscaleIP>
```
That's it. If your SSH server was set up correctly, you should be able to SSH using the command 
above. That's how easy it is to expose your SSH server using tailscale.

>Tip: Add an entry to your /etc/hosts file so that you don't have to remember the IP address 
> `echo "<tailscaleIP> <machine_name>" >> /etc/hosts` and then you can do 
> `ssh <username>@<machine_name>` .

#SSH using ngrok
ngrok is similar to tailscale, but it requires a bit more setup. You can install ngrok by following 
the instructions mentioned here. Once installed, you will have to start the local process that will 
redirect the network traffic. Run the following command to expose port 22 (which is the default port 
for SSH) once you have ngrok installed
```
ngrok tcp 22
```
This command will start a local process that will forward any requests ngrok receives on a specific 
IP:Port to this computer on port 22.

Once ngrok has started, go to this page https://dashboard.ngrok.com/status/tunnels. It should 
contain the hostname and the port you'll need to connect to your computer. Let's say the status page 
shows the following
```
tcp://0.tcp.ngrok.io:xxxxx
```
using this, you can ssh into the host computer as
```
ssh <username>@0.tcp.ngrok.io -p xxxxx
```

> Tip: ngrok has servers in many regions and your connection latency is determined by which server 
> you're connected to. The closer the server, the faster will be your connection and lower will be 
> the latency. You can find the list of regions here https://ngrok.com/docs#global-locations.
>
> My connection latency dropped by 10 times when I used a server located closer to my location. You 
> can set it as `ngrok tcp -region xx 22`

Manually running the ngrok binary might not be feasible always so you can create a service for ngrok 
which would start automatically every time the machine starts.

# Creating a service to start ngrok automatically
To allow ngrok to start automatically every time the computer starts, we'll create a service for it.
A service is a background process that can be configured to start automatically.

Create a new file in the following path
```
vi /etc/systemd/system/ngrok.service
```
and add the following to it
```
[Unit] Description=ngrok ssh tunnel
[Service] User=<your username>
ExecStart=<path to ngrok binary> tcp 22
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

Enable the new ngrok service
```
sudo systemctl enable ngrok
sudo systemctl start ngrok`
```
That's it. ngrok will now automatically start every time your computer starts. You're all set to use 
ngrok and SSH into your machine.
