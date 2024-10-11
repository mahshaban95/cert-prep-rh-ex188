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

