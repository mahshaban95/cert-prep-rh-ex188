# This directory will be used to follow the lessons and store the files related to the Oreilly Course: [Red Hat Certified Specialist in Containers (EX188)](https://learning.oreilly.com/course/red-hat-certified/9780135335956/)


## Firs step is to create your environment using DevBox

Check the DevBox Readme `../devbox.md`

## Lesson 1: Why use Containers

This lesson covers some concepts about containers and how Podman beats Docker (newer) and still provides compatibilty with it utilizing the OCI standatds.


## Lesson 2: Working with podman

### 2.1 Registry Access

```bash
## This needs developer redhat account (it is free)
podman login registry.redhat.io  

# Before that I had to create some confguration files to make it work on Ubuntu (not necessary if you are using CentOs or Rhel)

podman search ubi # Important as it gives you the FQCN (Fully Qualified Container Namepod) that is needed when pulling images from the RedHat registry (unlike docker images)

podman pull registry.redhat.io/ubi9/ubi
```

### 2.2 Running Containers

```bash
#### Important Podman Commands:
# podman login is used to log in on a specific registry.
# podman search searches for container images.
# podman pull fetches images from the registry and stores locally.
# podman images shows images stored by the current user.
# podman ps shows containers running for the current user.
# podman ps -a also shows containers no longer running.
# podman run pulls the container images, and runs it. Once the main application stops, the container stops.

podman images
podman run ubi
podman ps
podman ps -a
podman run ubi echo hello world
podman run nginx; Ctrl-C
podman run -d nginx
podman run -it ubi /bin/bash
```

### 2.3 Rootless versus Root Containers

- Unlike Root containers, Rootless containers does not get assigned an IP address. 
- If a rootless container needs to be accessed, you need to have port mapping. 
- The host os treats the rootless containers as just processes. You can see the processes in the host os using `ps faux |less` command and search for `conmon` which refers to container monitor. Check the child process for it to get more info about the containers running.

```bash
# Run an nginx container using podman from Docker's official image
podman run -d docker.io/library/nginx

# Check IP address information
sudo ip a

# Inspect the IP address of the last started container
podman inspect -l -f "{{.NetworkSettings.IPAddress}}"

# Run another nginx container using podman
sudo podman run -d nginx

# Check IP address information again
sudo ip a

# Inspect the IP address of the last started container again
sudo podman inspect -l -f "{{.NetworkSettings.IPAddress}}"

# Note: The '-l' flag refers to the last container that was started and avoids the need to specify a container name.
```

### 2.4 Managing Containers

```bash
# Shows current and past running containers
podman ps -a

# Stops a running container
podman stop <containername>

# Starts a stopped container again
podman start <containername>

# Shows properties used by containers and images
podman inspect <containername or image>

# Runs an additional shell as a secondary process in a container
podman exec -it <containername> sh

# Runs a shell as the primary process in a container (doesn't start the default command)
podman run -it <containername> sh

# Detaches from a current interactive session with a container
# Press Ctrl-p then Ctrl-q while inside the session
# You can always use a different detach keys
podman run -it --detach-keys="ctrl-x" <containername> sh

# Connects to a current interactive session on a container
podman attach <containername>

# This command will attempt to run the official MariaDB image but will fail
podman run docker.io/library/mariadb  # will fail

# Show the list of containers, including stopped ones
podman ps -a

# View the logs of the container to diagnose the failure
podman logs <containername>

# Correct way to run the MariaDB container by setting the MYSQL_ROOT_PASSWORD environment variable
podman run -d -e MYSQL_ROOT_PASSWORD=password mariadb
```

### 2.5 Container Networking

- Podman containers, by default, connect to the podman network.
- Use `podman network ls` to see current networks.
- To start containers in a different network namespace, use `podman network create` to create additional networks.
- Traffic between networks is blocked.
  - Remember also that rootless containers do not have IPs!


### 2.6 Accessing Containers

- Rootless containers cannot be exposed on privileged ports (i.e. 0-1024).

- In port forwarding, a source IP address can be specified to allow access only if traffic comes from a specified IP address.
  - Example: `podman run -d -p 127.0.0.1:8080:80 nginx`

```bash
# Run an nginx container, mapping port 8088 on the host to port 80 in the container
podman run -d -p 8088:80 nginx

# Test access to the exposed port
curl localhost:8088

# List running containers
podman ps

# Show the port mappings for the running containers
podman port <containername>
```

### 2.7 Restricting Containers

- Resource restrictions is implemented using cgroups on the Linux host.
- By default, cgroup restrictions do not work with rootless containers.

```bash
# Edit the containers.conf file to set PID limit to 0 (no limit)
sudo vim /etc/containers/containers.conf
# Add the following content under [containers]:
# [containers]
# pid_limit=0

# Create a directory for systemd service configuration
sudo mkdir /etc/systemd/system/user@.service.d

# Edit the delegate.conf file to delegate resources to the containers
sudo vim /etc/systemd/system/user@.service.d/delegate.conf
# Add the following content under [Service]:
# [Service]
# Delegate=memory pids cpu io

# Reload systemd to apply the changes
sudo systemctl daemon-reload

# Run a container with restricted memory (512MB) using UBI (Universal Base Image)
podman run -d -m 512M registry.redhat.io/ubi9/ubi sleep 1800

# Check the container statistics (including memory and CPU usage)
podman stats
```
