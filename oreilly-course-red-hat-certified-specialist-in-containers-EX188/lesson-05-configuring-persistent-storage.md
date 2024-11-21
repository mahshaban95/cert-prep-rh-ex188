# Lesson 5: Configuring Persistent Storage

## 5.1 Overlay Union Filesystems

### Understanding Container Storage

- Container images are composed of multiple layers, with each layer storing specific parts of the image efficiently.
- Each instruction in the Containerfile adds a new layer when building an image.
- Layers are immutable, so changes to earlier layers require rebuilding from that point onward.
- To minimize layers, it's recommended to consolidate commands, such as using only one `RUN` statement where possible.
- Use `podman image tree <imagename>` to investigate the different layers of an image.
- User modifications within a running container are written to a small, ephemeral read/write layer.

### Demo

```bash
# Display the layer structure of the "sleeper" image
podman image tree sleeper

# Display the layer structure of the "sleeper2" image
podman image tree sleeper2

# Run a container from the "sleeper2" image
podman run -d --name sleeper2 sleeper2

# Create a small file in the /tmp directory within the running "sleeper2" container
podman exec -it sleeper2 touch /tmp/smallfile

# List the contents of the container's overlay storage, replace <ID> with the container ID of "sleeper2"
ls ~/.local/share/containers/storage/overlay-containers/<ID>/userdata
```

## 5.2 Using Persistent Host Storage

### Providing Persistent Storage

- **Challenges with Writing to Container Filesystems**:
  - Writing modifications to the read/writable layer added on top of the container filesystem is inefficient:
    - Write operations are slow.
    - The read/writable layer is ephemeral.
    - Sharing the read/writable layer between containers is not possible.

- **Podman and Persistent Storage**:
  - Podman can directly access persistent host storage using:
    - **Volumes**: Podman-managed objects for persistent storage.
    - **Bind mounts**: User-managed data mounts.

- **Permissions**:
  - When accessing host storage, ensure the container UID has sufficient permissions.

#### Using Bind Mounts

- **Mount Options**:
  - Use the `-v` (`--volume`) option or the `--mount` option to mount persistent data.

- **Example Commands**:
  1. Create a directory for bind mount:
     ```bash
     mkdir /web
     ```
  2. Run a Podman container with a bind mount:
     ```bash
     podman run -p 8080:8080 --volume /web:/var/www/html:ro registry.access.redhat.com/ubi8/httpd-24:latest
     ```
  - The `--volume` option mounts the `/web` directory on the host to `/var/www/html` in the container in read-only mode (`ro`).

### Using Volumes

- Volumes are independent, Podman-managed objects for persistent storage.
- Create volumes with `podman volume create` (no extra args needed).
- Support local and NFS shared storage.
- Easily share volumes across containers.

### Using tmpfs Mounts

- **Purpose**: For ephemeral data that requires high performance and should not be stored in the container's CoW filesystem.
- **Mechanism**: Utilizes the host's `tmpfs` filesystem for temporary storage.
- **Example**:
  ```bash
  podman run --name=tmpfile --mount type=tmpfs,tmpfs-size=1G,destination=/tmp registry.redhat.io/ubi9/ubi dd if=/dev/zero of=/tmp/bigfile bs=1M count=1024
  podman logs tmpfile
  ```

### Importing and Exporting Data

- **Importing Data**:
  - Use `podman volume import` to load data from a tar archive into a volume.

- **Exporting Data**:
  - Use `podman volume export --output` to save data from a volume into a tar archive.

### Demo

```bash
# Create a Podman volume
podman volume create webdata

# Inspect the created volume
podman volume inspect webdata

# Create a sample HTML file
echo "hello world" > index.html

# Copy the HTML file to the Podman volume's storage path
cp index.html /home/student/.local/share/containers/storage/volumes/webdata/_data/

# Run a container using the volume and expose it on port 8083
podman run -d -p 8083:8080 --volume webdata:/app:Z docker.io/bitnami/nginx

# Test the application by curling the exposed port
curl localhost:8083
```

## 5.3 Managing Permissions

### Understanding Permissions

- Containers start in a new namespace with their own UID.
- The container UID must have permissions for the namespace.
- Check permissions with `podman unshare ls -ld /web`

```bash
# Create a directory for database files
mkdir /home/student/dbfiles

# Run a MySQL container with a volume and environment variable for the root password
podman run -d -v /home/student/dbfiles:/var/lib/mysql:Z -e MYSQL_ROOT_PASSWORD=password registry.access.redhat.com/rhscl/mysql-80-rhel7

# List details of the directory to verify permissions
ls -ld /home/student/dbfiles

# Inspect the container image to check the user information
podman inspect registry.access.redhat.com/rhscl/mysql-80-rhel7 | grep User

```

### Understanding Namespaces

- **Purpose**: Isolate files, processes, users, etc.
- **User Namespace**:
  - Non-root containers run in the same namespace.
  - UIDs are limited to the namespace; UID mappings are needed for external access.
- **Example**:
  - MySQL process runs as UID 27 inside the container namespace.
  - `/home/student/dbfiles` is outside the namespace, seen as `root:root` by the container.
  - Use `podman unshare ls -ld /home/student/dbfiles` to check permissions as seen by the container.
  - Use `podman unshare chown 27:27 /home/student/dbfiles` to change ownership.

## 5.4 Managing SELinux Settings

- **SELinux Access Control**:
  - Containers use the `container_t` context by default.
  - Mounted directories must have the `container_file_t` context.

- **Ensuring Proper Context**:
  - Use the `:Z` option with the `-v` flag to automatically set the correct SELinux context:
    ```bash
    podman run -v /web:/var/www/html:Z
    ```
  - This assigns the proper SELinux context (`container_file_t`) and creates a unique category label for the container.

- **Benefits**:
  - SELinux categories ensure containers can only access their designated storage.
  - Podman automatically handles SELinux configuration for volumes.

- **Note**:
  - When using Podman volumes, SELinux configuration is managed automatically.


## 5.5 Configuring Storage for Stateful Applications

- **Stateless vs Stateful**:
  - Web-based applications are typically **stateless** and do not maintain state.
  - Databases are **stateful** and require data persistence.

- **Challenges of Stateful Applications in Containers**:
  1. When moved to a different environment, data must remain accessible in the new environment.
  2. Running multiple database instances requires synchronization to prevent data conflicts.
  3. Achieving high availability for databases is more complex.

- **Key Requirement**:
  - Stateful containers must always have access to the same data, regardless of where they are running.

### Best Practices

- **Use Volumes**:
  - Mount persistent data on volumes instead of bind mounts.

- **VOLUME Instruction**:
  - Utilize the `VOLUME` instruction in `Containerfile` to define and create volumes.

- **Dedicated Network**:
  - Create a dedicated network to enable seamless communication between client and server processes.

### Demo

```bash
# Create a volume for MySQL data
podman volume create mysqldata

# List all volumes to verify
podman volume list

# Create a dedicated network
podman network create datanet

# List all networks to verify
podman network ls

# Run a MariaDB container with persistent storage and a custom network
podman run -d --name persistent-mysql \
  -e MARIADB_ROOT_PASSWORD=password \
  -v mysqldata:/var/lib/mysql \
  --network=datanet docker.io/bitnami/mariadb
```