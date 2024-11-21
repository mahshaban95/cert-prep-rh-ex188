# Lesson 7: Integrating Podman Containers with Systemd

## 7.1 Systemd User Units

### Why Systemd?

- Podman uses the fork/exec model to run containers as Linux commands.
- Any process running as a Linux command can be managed using the default service manager, **Systemd**.
- Systemd manages daemon processes through service unit files.
- Podman containers can start as user processes or daemon processes via Systemd.


### Auto-Restarting Units

- Podman can automatically restart containers.
- Use `--restart=always` while starting a container to ensure it restarts automatically.
- To configure the restart policy on system boot:
  ```bash
  systemctl enable --now podman-restart # Root Level
  systemctl --user enable --now podman-restart # User Level
  ```

### Creating Systemd User Units

- Any user command can be started as a Systemd user unit file.
- Place unit files in ~/.config/systemd/user.
- Enable user unit files with `systemctl enable --user <unit-name>`
- To auto-start a user unit file without requiring the user to log in, enable lingering `loginctl enable-linger <username>`
Copy code

### Demo

```bash
# Create a new user "anna" and set a password
sudo useradd anna
sudo passwd anna

# Enable linger for the user "anna" to allow services to run even when the user is not logged in
sudo loginctl enable-linger anna

# Switch to the user "anna" via SSH or su
ssh anna@localhost

# Create the directory for user-specific systemd services and navigate to it
mkdir -p ~/.config/systemd/user
cd ~/.config/systemd/user

# Create a new service file named "sleep.service"
vim sleep.service

# Inside the "sleep.service" file, add the following content:
# [Unit]
# Description=Sleep Service
#
# [Service]
# Type=simple
# ExecStart=/bin/sleep infinity
#
# [Install]
# WantedBy=default.target

# Enable and start the service
systemctl --user enable --now sleep.service

# Verify the status of the service
systemctl --user status sleep.service
```

## 7.2 Generating Systemd User Units


- The `podman` command can generate user unit files for running containers.
- To do so, the actual user should be in the directory `~/.config/systemd/user`.
- Next, use the following command `podman generate systemd --name containername -n --new --files`
  - Use `--files` to generate a service file.
  - Use `-n` to create the service file by name and not container ID.
  - Add `--new` to stop the currently running container and create a new one.

```bash
# Run these commands as the user "anna" created in the previous demo

# Navigate to the systemd user directory
cd ~/.config/systemd/user

# Run a Podman container with a specific name and port mapping
podman run -d -p 8082:8080 --name annanginx docker.io/bitnami/nginx

# Generate a systemd unit file for the container
podman generate systemd annanginx --files -n

# Reload the systemd user daemon
systemctl --user daemon-reload

# Check the status of the container's systemd service
systemctl --user status container-annanginx

# Enable the container's systemd service to start on boot
systemctl --user enable container-annanginx

# Reboot the system
sudo reboot

# As a non-anna user, check the process for nginx
ps faux | grep -C5 nginx

```

## 7.3 Automatically Updating Systemd Managed Containers


- When Podman containers are managed by Systemd, they can be updated automatically.
- To do so, the container must have a label set to `io.containers.autoupdate=registry`.
- Use the following command to enable daily automatic updates `sudo systemctl enable --now podman-auto-update.timer`
- Alternatively, use the following command to trigger the update process manually `podman auto-update`
- To roll back to the previous version if the update fails, use `podman auto-update --rollback`

```bash
# Run a Podman container with the auto-update label
podman run --label "io.containers.autoupdate=registry" -p 8083:8080 --name autonginx docker.io/bitnami/nginx

# Navigate to the systemd user directory
cd ~/.config/systemd/user

# Generate a systemd service file for the container
podman generate systemd -n --files --new autonginx

# Stop the container
podman stop autonginx

# Enable and start the container as a systemd service
systemctl --user enable --now container-autonginx

# Trigger a manual auto-update
podman auto-update

# Enable the Podman auto-update timer for daily updates
sudo systemctl enable --now podman-auto-update.timer
```