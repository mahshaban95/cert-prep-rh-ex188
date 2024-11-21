# Lesson 6: Troubleshooting Containers

## 6.1 Container Logs

- Use `podman ps [-a]` to check the status of containers.
- Logs include application output and errors, stored in the read/writable layer that is added while the container is running.
- Access logs for any container (running or stopped) with `podman logs <container_name_or_id>`

```bash
# Attempt to run a MariaDB container without required environment variables
podman run --name faildb -d docker.io/bitnami/mariadb

# List all containers to check its status
podman ps -a

# Check logs for troubleshooting
podman logs faildb

# Remove the failed container
podman rm faildb

# Correctly run the container with required environment variables
podman run --name faildb -d -e MARIADB_ROOT_PASSWORD=password docker.io/bitnami/mariadb
```

### Networking Issues

- **Port Mapping**:
  - Containers are accessed by mapping a host port to the container's application port.

- **Troubleshooting Host Port Access**:
  - If the host port isn't working, check the port the application is running on:
    ```bash
    podman exec -it <CONTAINER> ss -pant
    ```

- **Internal Network Containers**:
  - Not all containers expose ports externally. Some connect only to internal networks.

- **Finding the Network ID**:
  - Use the following command to inspect the container's network details:
    ```bash
    podman inspect <CONTAINER> --format='{{.NetworkSettings.Networks}}'
    ```

## 6.2 Connecting to Running Containers

### Connecting to Containers
- **Use `podman exec`**:
  - Connect to a running container and execute commands available in the container image.
  - Example:
    ```bash
    podman exec -it <container_name_or_id> <command>
    ```

- **For Missing Commands in the Container**:
  - Use `sudo nsenter` to run commands from the host within the container namespace.
  - Example:
    ```bash
    sudo nsenter -t <container_pid> -n <command>
    ```

### Running Host Commands Within Containers
- Containers may lack many commands needed for debugging or operations.
- **Find the Container PID**:
  - Use the following command to retrieve the PID of the container:
    ```bash
    podman inspect <container_name_or_id> --format '{{.State.Pid}}'
    ```

- **Run Commands in the Container Namespace**:
  - Use the `nsenter` command with the container PID:
    ```bash
    sudo nsenter -t <container_pid> -n <command>
    ```
### Demo 
```bash
# Run an NGINX container with port mapping
podman run -d -p 8080:80 --name bitweb docker.io/bitnami/nginx

# Check the mapped ports for the container
podman port bitweb

# Test the connection using curl (this may fail if the container isn't running as expected)
curl localhost:8080

# Check active connections inside the container (may fail if no connections exist)
podman exec -it bitweb ss -pant

# Retrieve the container's PID
export BITWEBPORT=$(podman inspect bitweb --format '{{.State.Pid}}')

# Use nsenter to access the container's namespace and check connections
sudo nsenter -n -t $BITWEBPORT ss -pant
```

## 6.3 Troubleshooting Network Issues


- Containers can be connected to a network using the `--net <networkname>` option when running a new container.

- To show the networks a container is using:
  ```bash
  podman inspect <containername> --format='{{.NetworkSettings.Networks}}'
  ```

- To connect to a running container t a network:
  ```bash
  podman network connect <networkname> <containername>
  ```
- Network are namespaced just as like containers are namespaced. So, `podman network ls` is different from `podman network ls`.