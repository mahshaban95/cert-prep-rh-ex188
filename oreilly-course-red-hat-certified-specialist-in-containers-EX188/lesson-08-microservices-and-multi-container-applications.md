# Lesson 8: Microservices and Multi-container Applications

## 8.1 Understanding Microservices

- A microservice is an application that consists of different independent components that are connected together.
- The independent components can be individually developed by completely disconnected projects.
- An additional layer is used that adds site-specific information to the microservices.

### Why Use Microservices?
- Developing independent, small applications is easier than maintaining one big application.
- Changes can be applied faster.
- By applying common DevOps technologies like Continuous Integration/Continuous Development, zero-downtime application upgrades can be provided.

### Microservices Benefits
- When broken down in pieces, applications are easier to build and maintain.
- Each piece is independently developed, and the application itself is just the sum of all pieces.
- Smaller pieces of application code are easier to understand.
- Developers can work on applications independently.
- If one component fails, you spin up another while the rest of the application continues to function.
- Smaller components are easier to scale.
- Individual components are easier to fit into continuous delivery pipelines and complex deployment scenarios.

### Understanding Containers and Microservices
- A container is an application that runs based on a container image. The container image is a lightweight, standalone, executable package of software that includes everything that is needed to run an application.
- Because containers are lightweight and include all dependencies required to run an application, containers have become the standard for developing applications.
- As containers are focusing on their specific functionality, they are perfect building blocks for building microservices.
- In a containerized, microservices environment, site-specific information can be provided in a flexible way by using orchestration tools such as Kubernetes.


## 8.2 Using Compose to Run and Manage Microservices

### Using Microservices in Podman

- **podman-compose** uses a YAML file (`compose.yaml`) that defines one or more containers with their required properties.
- This allows you to run connected containers in a development environment.
- **podman-compose** is not the recommended way to run connected containers in production; it is better to use Kubernetes or OpenShift.
- As an alternative, Podman can use Pods, which are based on the way containers are started through Kubernetes YAML files.
- Use `podman generate kube` to generate these YAML files from existing containers.
- Use `podman play kube` to create containers based on Kubernetes YAML files.

### Using podman-compose

- To use **podman-compose**, you'll first have to install it (not required in the exam):
  - `sudo subscription-manager repos --enable codeready-builder-for-rhel-9-x86_64-rpms`
  - `sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm`
  - `sudo dnf upgrade`
  - `sudo dnf install podman-compose`
- Write a `compose.yaml` file with the instructions you want to execute.
- Use `podman-compose up` to process the instructions in the compose file and start all services specified.
- To start all containers in detached mode, use `podman-compose up -d`.

### Understanding Compose.yaml

- In a `compose.yaml` file, different resources are created:
  - **services**: These are the containers with all of their parameters. Multiple services can be created as child elements to the services parameter.
  - **volumes**: Allows for automated creation of storage volumes.
  - **networks**: Allows for the creation of dedicated networks.
- To specify container properties, use the following:
  - **image**: The complete reference to the image you want to use.
  - **networks**: The networks to which you want the container to connect.
  - **ports**: Port mappings (equivalent to the `podman run -p` option).
  - **environment**: Environment variables you'd like to set.

---

### Example Compose File

```yaml
services:
  frontend:
    image: frontend:latest
    ports:
      - "8080:80"
    networks: datanet
    volumes:
      - datavol:/data:Z
  backend:
    # Additional configurations for backend service
networks:
  datanet: {}
volumes:
  datavol: {}
```