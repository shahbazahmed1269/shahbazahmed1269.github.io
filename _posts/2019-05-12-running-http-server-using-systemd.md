## Running HTTP server in EC2 using Systemd

### Problem
So today while deploying a Golang HTTP server in EC2 instance with Amazon AMI (built on top of Fedora distro of Linux OS), I hit an interesting roadblock. I copied the Go executable into the server and executed it over SSH and everything worked fine. But when I closed the SSH connection the website went down. I had deployed web servers in Docker based environments but this was the first time I was setting up a VM. Then realized that I needed it to run as a service daemon, so that it will be spawned even if the server crashed or the EC2 instance restarted. 

### One Possible Solution
Upon quick googling I stumbled upon Linux service manager called `systemd`. As an added bonus Fedora comes with `systemd` installed by default, so I thought lets use it.

### Systemd Intro

Wikipedia page describes `systemd` as:
> The systemd software suite provides fundamental building blocks for a Linux operating system. It includes the systemd "System and Service Manager", an init system used to bootstrap user space and manage user processes.

On each Unix system, there is one process called `init` process or PID 1, which is the first process started by the kernel.It is the direct or indirect parent of all processes, and it replaces the parent of a process when the original parent terminates. Due to this, it can do lot of things that other process can't such as spawning up and maintaining userspace during boot and is well suited for the purpose of monitoring daemons.  

`systemd` runs as a PID 1 and is responsible of starting and managing services and daemons, logging, running containers and VMs, and lots more. `systemd` performs its startup tasks in parallel and uses unix-sockets and d-bus for IPC communication.

The core components of `systemd` include:
1. `systemd`: is the Linux service manager 
2.  `systemctl`: is basically a part of `systemd` which provides ability to interact with and control the state of `systemd` service manager. Linux `service` command is a wrapper around systemctl command depending the OS distro.
3.  `systemd-analyze`: used to analyze the performance statistics and tracing information retrieved from `systemd` service manager.

Other components:<br>
`systemd` suite provides other components such as `journald` (event logging and log file daemon), `logind` (daemon for managing user logins), `timedated` (daemon for controlling datetime settings) and many more. 

### Cheatsheet

1. `systemctl start [name.service]`:  start a service
2. `systemctl stop [name.service]`:  stop a service
3. `systemctl restart [name.service]`: restart a service
4. `systemctl reload [name.service]`: reload the service config from its `.service` file
5. `systemctl status [name.service]`: check the status of the service
6. `systemctl is-active [name.service]`: check whether the service is active or not.
7. `systemctl list-units --type service --all`: lists all active services

### Demo
1. I built the Go server executable for linux environment using command `env GOOS=linux GOARCH=amd64 go build -o hello-web` from my mac.
2. Create a file called `hello-web.service` in path `/etc/systemd/system/hello-web.service` having following content:

```
[Unit]
Description=hello-web

[Service]
Environment=SYSTEMD_LOG_LEVEL=debug 
Type=simple
Restart=always
RestartSec=5s
WorkingDirectory=/home/ec2-user/projects/hello-web
ExecStart=/home/ec2-user/projects/hello-web/hello-web -debug=true
```

Here `hello-web` is a Go executable for an HTTP server that we want to run as a daemon. `WorkingDirectory` defines on which directory the service will be launched.  

<b>NOTE: </b> Not providing `WorkingDirectory` may lead to unexpected errors and consequences such as relative path not working as expected. I learnt it the hard way.

3. Execute `sudo systemctl start hello-web.service`. `.service` can be ignored as its the default extension while working with systemctl. This command is equivalent of `sudo service hello-web start`
4. After making any changes to service file `hello-web.service`, we must run `sudo systemctl daemon-reload`
5. To check the status of the service, execute `systemctl status hello-web.service`
6. In case the service deoesn't start, get the PID of the failed process and use `journald` to check the error logs: `journalctl _PID=<YOUR_PID>`. This is how I got to know about relative path issue which was fixed by setting `WorkingDirectory`in service file.

### Ending note 
I have almost always deployed my service using services like <b>Docker</b>, <b>Kubernetes</b> and other container platforms. But I today learnt something
new way to deploy services using EC2 and `systemd`. I will play around to learn more about `systemd` and similar tools. If you have any suggestions or improvements, it is more than welcome.

### Resources
1. [Wikipedia page](https://en.wikipedia.org/wiki/Systemd)
2. [Digital Ocean tutorials: Systemd essentials](https://www.digitalocean.com/community/tutorials/systemd-essentials-working-with-services-units-and-the-journal)
3. [Medium blog: Creating a Linux service with systemd](https://medium.com/@benmorel/creating-a-linux-service-with-systemd-611b5c8b91d6)