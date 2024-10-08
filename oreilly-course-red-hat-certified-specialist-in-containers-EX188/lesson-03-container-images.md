## Lesson 3: Container Images

### 3.1 Using Image Registries

- In Podman, there is no default registry. You must indicate fully qualified image names like `docker.io/library/nginx` and not just `nginx`.
- A list of unqualified search registries can be provided through `/etc/containers/registries.conf`.
- Example of Image registries:
  - `docker.io`: the biggest registry for OCI standards.
  - `registry.access.redhat.com`: for Red Hat public registries, optimized for Podman and RH OpenShift.
  - `registry.redhat.io`: requires Red Hat developer account.
  - `quay.io`: community registry sponsored by Red Hat.

```bash
podman run -d nginx # choose docker.io 
podman images
podman run -d nginx # now works from local image
podman run docker.io/library/busybox
grep 'unqualified-search' /etc/containers/registries.conf
```

### 3.2 Pulling Images

- Images are composed of different storage layers.
- Layers are fetched separately and then combined together to work as a runnable image.
- You can prefetch images with the `podman pull` command.
- By default, if an image exists locally it is not fetched from the remote registries unless you use `podman run --pull=<option>` where option is `always`,`never`, `newer` or `missing`.

```bash
podman run --pull=never -d docker.io/library/mariadb
podman run --pull=newer -d docker.io/library/mariadb
podman image list
podman image inspect mariadb[Tab]
podman image tree mariadb[Tab] # shows all layers
```

### 3.3 Managing Images

- In Podman, images are stored separately for each user which is inefficient but more secure.

```bash
# List available images
podman image list

# Show details of a specific image
podman image inspect <image>

# Remove a specific image
podman image rm <image>

# Remove all unused images
podman image prune # Should be a cron job
```

### 3.4 Creating Custom Images from Running Containers

- Custom images can be created in different ways:
  - Build from scratch using Containerfile.
  - Create from changes made to a running container.
  - Create by using advanced tools such as `buildah`. (Not required for EX188)
  
- To save changes that are made inside a running container to an image, use `podman commit`.

```bash
# Run an nginx container with an interactive shell and name it "customweb"
podman run --name customweb -it nginx sh

# Inside the container, create a file and then exit
touch /tmp/testfile
exit

# Commit the changes made to the "customweb" container into a new image called "nginx:custom"
podman commit customweb nginx:custom

# List available images to verify the newly created image
podman images

# Run a container from the custom image and check for the test file
podman run -it localhost/nginx:custom ls -l /tmp/testfile
```

### 3.5 Using Tags

```bash
podman image tag docker.io/library/nginx mynginx:1.0podman image tag docker.io/library/nginx mynginx:1.0
podman image list
# Note that the image ID is the same, i.e. an image is not copied. It is the same image with two names now.
```