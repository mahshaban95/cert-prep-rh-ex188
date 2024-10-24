## Lesson 4: Creating Custom Container Images

### 4.1 Using Containerfile

- There are no difference between the words `Containerfile` and `Dockerfile` in functionality but Red Hat prefers the word `Containerfile`.
- Each instruction in a `Containerfile` results in an image layer added to the final image.
- In a `Containerfile`, `FROM` refers to the base images. Examples of base images are `alpine`, `busybox` and `UBI`.
- Universal Base Image (UBI): Provided by Redhat and exists in different versions:
  - standard: essential Linux utilities.
  - init: uses `systemd` to simplify running multiple apps on one container.
  - minimal: uses the `microdnf` package manager for efficiency.
  - micro: smallest UBI, doesn't include a package manager.

- Common Containerfile Instructions:
  - **FROM**: Identifies the base image to use.
  - **WORKDIR**: Sets a working directory for all next instructions.
  - **COPY**: Copies files from the host to the image.
  - **ADD**: Copies files from host or URL, or unpacks files from tar archives. More flexible than `COPY`.
  - **RUN**: Runs a command while building the image. Files created by using `RUN` exist in the final image created.
  - **ENTRYPOINT**: Defines the default command to use. Commonly set to a shell.
  - **CMD**: Specifies a command to run while starting the resulting image.
  - **USER**: Defines the user account that runs the container commands.
  - **LABEL**: Defines a key-value pair used for informational purposes.
  - **EXPOSE**: Shows which port is exposed by the container (without actually exposing it).
  - **ARG**: Defines arguments that can be used while building the image from the Containerfile.
  - **ENV**: Defines environment variables for use in the container.
  - **VOLUME**: Specifies where to store data outside of the container.

- To build the image, run the command `podman build -t <image-name> .` where `.` is the directory where the `Containerfile` is located.

- Here is a simple Containerfile:
```yaml
# Needs authentication to the Red Hat registry
FROM registry.redhat.io/ubi9/ubi
# Needs Red Hat system registrtaion (alternatively, you can just use CentOS)
RUN dnf install -y nmap
CMD echo hello world
```

### 4.2 Applying Best Practices to Create a Containerfile

- Use a base image that contains only what you need.
- Define a `WORKDIR` to set the working environment for the container.
- Avoid unnecessary `RUN` statements, as each adds another layer to the image.
- Use labels to provide additional information about the image.
- See examples of best practices [here](https://github.com/sandervanvugt/ex188).
  - Note: the examples in this repo use CentOS which might cause package issues, I used Rocky Linux instead and it worked as expected.
- To build the `Containerfile` from the command line, run `podman build -t <image-name> .`

### 4.3 Advanced Containerfile Options

- Multistage builds:
  - Containerfile support for multistage builds. This is important because it allows you to not include build tools in the final image, which leads to a smaller image size.
  - An example of using multistage is in `./multistage-containerfile/Containerfile`
  - Use `podman image tree <image-name>` to see the multiple stages of your image.

- Volumes:
  - Volumes are used to provide persistence storage for containers.
  - Using the `VOLUME` instruction in the Containerfile does NOT actually create a volume when building the image. You need to actually run the the container that mounts a volume on a specified directory.

```bash
# Build the container image using a Containerfile
sudo podman build -t countdown ./countdown-containerfile/Containerfile

# Run the container with a volume mounted at /mydata
sudo podman run -v countdown:/mydata countdown

# List all the volumes
sudo podman volume list

# Inspect a specific volume (use Tab to autocomplete the volume name)
sudo podman volume inspect <volume-name>

# List the contents of the volume data directory
ls -l /var/lib/containers/storage/volumes/countdown/_data/
```

- CMD vs. ENTRYPOINT
  - Containers start a command specified as either CMD or ENTRYPOINT. Commands defined in CMD can be overridden when running the container, while those in ENTRYPOINT cannot be replaced but can accept additional arguments.
  - Using `podman run -it nginx sh` overrides the CMD, starting a shell instead. In contrast, `podman run countdown 2` specifies an argument for the ENTRYPOINT command without changing it.
    - `podman run -it nginx sh` works because the entrypoint is a special script that make is easy to provide the command `sh` as an argument to that script. To verify this, check the ENTRYPOINT abd CMD section in `podman inspect nginx`

### 4.4 Containerfile ARG and ENV

- Environment variables can be used while building container images.
- These variables must be declared in the Containerfile using `ARG`.
  - Example: `ARG VERSION="1.2"`
- Alternatively, the `--build-arg KEY=VALUE` option can be used while building to define variables.
  - Example: `podman build --build-arg version="1.2" -t versionapp .`

```bash
# Create a Containerfile with the following content
cat <<EOF > Containerfile
FROM ubi9/ubi
ARG arguser
ENV user=\${arguser}
RUN mkdir /test && \
    touch /test/\${user}
CMD ls -l /test/
EOF
# Note that you can not use the ARG directly. You first need to pass it as a variable using ENV


# Build the container image using the build argument "arguser"
podman build --build-arg arguser=anna -t argtest .
```

### 4.5 Managing Container User IDs


- **Rootless Containers**:
  - Podman containers can be started as rootless containers.
  - The containerized process does not use the host's Linux root user.
  - The root user inside the container is mapped to a limited user outside the container.
  - The container runtime does not need to be started by the root user.
  - When Podman containers are started by non-privileged users, they are rootless by default.
  - If the root account is used inside the container, it is mapped to a non-root user on the host OS.
  - It is best practice to run non-root users within the containers as well.

- **Default User Behavior**:
  - Many images run as the root user by default.
  - In `Containerfile`s, if no user account is specified, the default image user account is used.
  - After using `RUN useradd` within the `Containerfile`, the newly added user can be set as the standard user.
  - It is safer to use non-root users within containers even in restricted environments, as a root user inside could be a security risk.

- **User Mappings in Rootless Containers**:
  - When non-root users start a container, the user inside the container is mapped to a user on the host OS.
  - Container user ID 0 (root) maps to the UID of the user that started the container on the host.
  - Other users inside the container are mapped to `ID + 99999` on the host.
    - Example: A MySQL user with UID 27 in the container would get UID 100026 on the host.
    - Required host users are dynamically created.
  - The `/etc/subuid` and `/etc/subgid` files contain the name of the current user, the starting dynamic UID on the host (typically 100000), and the number of available UIDs (typically 65536).
  - To verify user mappings for currently running containers, use:
    ```bash
    podman top <containerID> user user
    ```

- **Demo**:

```bash
# Step 1: Create a Containerfile with the following content
cat <<EOF > Containerfile
FROM registry.access.redhat.com/ubi9/ubi
CMD ["sleep", "3600"]
EOF

# Step 2: Build the image
podman build -t sleeper .

# Step 3: Run the container
podman run -d --name sleeper sleeper

# Step 4: Check the user and group mappings inside the container
podman top sleeper huser user

# Step 5: Add the following to the Containerfile to create a new user
cat <<EOF >> Containerfile
RUN useradd sleeper
USER sleeper
EOF

# Step 6: Build the updated image
podman build -t sleeper2 .

# Step 7: Run the updated container
podman run -d --name sleeper2 sleeper2

# Step 8: Check the user and group mappings inside the updated container
podman top sleeper2 huser user
```

- **Rootless Container Limitations**:
  - Some applications cannot easily run as a rootless container.
  - Apache, for instance, starts a main process with root privileges, which then spawns child processes that run as a non-privileged user.
  - For applications like this, Red Hat provides customized images that run as a non-privileged user.
  - Bitnami is also a common provider of rootless images.
  - Rootless containers cannot bind to a privileged port.
  - Some utilities (like `ping`) cannot be used from within a rootless container.

### 4.6 Hosting a Private Registry

- A private registry can be created for easier and more secure access to images, especially those meant for internal use only.
- Private registries provide a secure location for internal images, reducing the need for public repositories.
- To run a private registry, use the registry container.
- To upload images to a private registry, you must first tag the image with the registry location:
  ```bash
  podman tag fedora:latest localhost:5000/myfedora:latest
  ```

- Public registries should use TLS certificates to ensure secure access. However for private registries and internal use, it may be acceptable to allow insecure access.
- To configure a registry for insecure access, create a file `/etc/containers/registries.conf.d/myregistry.conf` with the following content:
  ```
  [[registry]]
  location = "localhost:5000"
  insecure = true
  ```
- Alternatively, you can disable TLS verification when pushing an image by using the `--tls-verify=false` option:
  ```bash
  podman push --tls-verify=false localhost:5000/myfedora:latest
  ```` 

- **Demo:**
```bash
# Enable and start the podman-restart service to restart containers that are flagged for restart
sudo systemctl enable --now podman-restart.service

# Run a private registry container with port 5000 exposed and set to always restart
podman run -d -p 5000:5000 --restart=always --name registry docker.io/library/registry

# Pull the Fedora image
podman pull fedora

# List available images
podman images

# Tag the Fedora image for the private registry
podman tag fedora:latest localhost:5000/myfedora

# Push the tagged image to the private registry (with insecure TLS verification disabled)
podman push localhost:5000/myfedora --tls-verify=false

# Remove the local Fedora image
podman rmi fedora

# Verify the image in the registry by executing a shell inside the registry container and searching for the image
podman exec -it registry sh -c "find . -name 'myfedora'"

# Pull the image from your private registry
podman pull localhost:5000/myfedora --tls-verify=false
```