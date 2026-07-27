# Lab 06 — Docker Networking & Bridge Networks

## 🎯 Objective

Create custom **Docker bridge networks** and enable **container-to-container DNS resolution**.

By the end of this lab, you will understand how Docker containers communicate across custom networks and how Docker's embedded DNS allows containers to communicate using **container names** instead of IP addresses.

---

# 🌐 Docker Networking Fundamentals

Docker networking provides connectivity between containers, the Docker host, and external networks.

A custom bridge network can be visualized as:

```text
                  Docker Host
                       │
                ┌──────┴──────┐
                │ devops-net  │
                │   Bridge    │
                └──────┬──────┘
                       │
            ┌──────────┴──────────┐
            │                     │
        Container A          Container B
        Name: app             Name: db
            │                     │
            └────── DNS ──────────┘
```

Containers connected to the same custom bridge network can communicate using their **container names**.

For example:

```text
app → db
```

instead of requiring the IP address of the `db` container.

---

# 🔧 Networking Commands

## List Docker Networks

```bash
docker network ls
```

Lists all Docker networks available on the host.

---

## Create a Custom Bridge Network

```bash
docker network create devops-net
```

Creates a custom bridge network named:

```text
devops-net
```

---

## Inspect a Network

```bash
docker network inspect devops-net
```

Displays detailed information about the network, including:

* Network configuration
* Subnet
* Gateway
* Connected containers
* Container IP addresses

---

# 🐳 Run Containers on the Same Network

Run an Nginx container:

```bash
docker run -d --name app --network devops-net nginx
```

Run a MySQL container:

```bash
docker run -d \
  --name db \
  --network devops-net \
  mysql:8 \
  -e MYSQL_ROOT_PASSWORD=root123
```

> ⚠️ **Important:** For Docker images that use environment variables to configure startup, the environment variable option is conventionally placed before the image name. A corrected form is:

```bash
docker run -d \
  --name db \
  --network devops-net \
  -e MYSQL_ROOT_PASSWORD=root123 \
  mysql:8
```

---

## 🔗 Container-to-Container Communication by Name

Containers on the same custom network can communicate using their **container names through Docker's embedded DNS**.

For example:

```bash
docker exec -it app ping db
```

The `app` container can resolve:

```text
db
```

to the IP address assigned to the MySQL container.

> 💡 **Key Concept:** This is one of the major advantages of custom Docker networks. Applications can communicate using stable service/container names rather than hard-coded container IP addresses.

---

# 🔌 Connect an Existing Container

Connect an existing container to a network:

```bash
docker network connect devops-net mycontainer
```

---

# 🔌 Disconnect a Container

Disconnect an existing container:

```bash
docker network disconnect devops-net mycontainer
```

---

# 🗑️ Remove a Network

```bash
docker network rm devops-net
```

> ⚠️ **Caution:** A Docker network generally cannot be removed while containers are still connected to it. Disconnect or remove the containers first.

---

# 🟢 Task Set 1 — Guided

| Task     | What to do                                                                        |
| -------- | --------------------------------------------------------------------------------- |
| **T1.1** | Create network: `docker network create lab-net`. Run nginx and curl in `lab-net`. |
| **T1.2** | From curl container: `curl http://nginx` (resolve by name). Observe DNS working.  |
| **T1.3** | Inspect the network: `docker network inspect lab-net` — find both container IPs.  |

---

## T1.1 — Create a Custom Network

Create the network:

```bash
docker network create lab-net
```

Verify it:

```bash
docker network ls
```

Run an Nginx container on the network:

```bash
docker run -d \
  --name nginx \
  --network lab-net \
  nginx
```

Run a second container that can be used to execute `curl` commands:

```bash
docker run -d \
  --name curl \
  --network lab-net \
  alpine \
  sleep 3600
```

Enter the `curl` container:

```bash
docker exec -it curl sh
```

> 💡 **Note:** The minimal Alpine image may not include `curl` by default. If required, install it inside the container:

```bash
apk add --no-cache curl
```

---

## T1.2 — Test DNS Resolution

From inside the `curl` container, run:

```bash
curl http://nginx
```

Docker's embedded DNS should resolve:

```text
nginx
```

to the IP address of the Nginx container.

You should receive the Nginx HTTP response.

Exit the container:

```bash
exit
```

The communication path is:

```text
curl container
      │
      │ DNS: nginx
      ▼
Docker Embedded DNS
      │
      ▼
nginx container
      │
      ▼
HTTP :80
```

---

## T1.3 — Inspect the Network

Run:

```bash
docker network inspect lab-net
```

Locate the container information and identify the IP addresses assigned to:

```text
nginx
curl
```

Observe that both containers are members of:

```text
lab-net
```

---

# 🟡 Task Set 2 — Practice

| Task     | What to do                                                                                     |
| -------- | ---------------------------------------------------------------------------------------------- |
| **T2.1** | Create 2 networks: `frontend-net` and `backend-net`. App connects to both, DB only to backend. |
| **T2.2** | Verify isolation: can frontend container reach DB directly? It should **NOT** be able to.      |
| **T2.3** | Use `--network host` mode on nginx. Verify nginx is accessible on host port 80 directly.       |

---

## T2.1 — Create Frontend and Backend Networks

Create two separate networks:

```bash
docker network create frontend-net
docker network create backend-net
```

The desired architecture is:

```text
                    frontend-net
                         │
                         ▼
                       App
                    /       \
                   /         \
                  ▼           ▼
         frontend-net     backend-net
                               │
                               ▼
                              DB
```

The intended connectivity is:

```text
App → Frontend Network
App → Backend Network
DB  → Backend Network only
```

Run the application container:

```bash
docker run -d \
  --name app \
  --network frontend-net \
  nginx
```

Connect the same application container to the backend network:

```bash
docker network connect backend-net app
```

Run the database only on the backend network:

```bash
docker run -d \
  --name db \
  --network backend-net \
  -e MYSQL_ROOT_PASSWORD=root123 \
  mysql:8
```

Inspect the networks:

```bash
docker network inspect frontend-net
docker network inspect backend-net
```

Confirm that:

* `app` is connected to both networks.
* `db` is connected only to `backend-net`.

---

## T2.2 — Verify Network Isolation

The goal is to verify that a frontend-only container cannot directly reach the database.

The intended topology is:

```text
Frontend Container
       │
       │ ✕
       ▼
      DB

App Container
       │
       │ ✓
       ▼
      DB
```

Inspect network membership:

```bash
docker network inspect frontend-net
docker network inspect backend-net
```

Confirm that the database is **not connected to `frontend-net`**.

> ⚠️ **Important:** Docker network segmentation provides network-level isolation, but it should not be treated as a complete security policy by itself. Production environments should also use authentication, authorization, firewall controls, and application-level security.

---

## T2.3 — Use Host Networking

Run Nginx using host networking:

```bash
docker run -d \
  --name nginx-host \
  --network host \
  nginx
```

With host networking, the container uses the host's network namespace rather than a separate Docker bridge network.

Because Nginx listens on port `80`, test it from the EC2 host:

```bash
curl http://localhost:80
```

You can also test:

```bash
curl http://<EC2-IP>:80
```

> ⚠️ **Important:** With `--network host`, Docker port publishing such as `-p 8080:80` is not used in the normal bridge-network sense. The application binds directly to the host network.

> 💡 **Note:** Host networking behaves differently depending on the operating system and Docker environment. This exercise is best performed on a Linux EC2 host.

---

# 🔴 Task Set 3 — Challenge

| Task     | What to do                                                                                          |
| -------- | --------------------------------------------------------------------------------------------------- |
| **T3.1** | 3-tier setup: Nginx (frontend) → Flask app (backend) → Redis (cache). All on separate segments.     |
| **T3.2** | Write `docker-compose.yml` for above 3-tier setup with named networks. Verify communication.        |
| **T3.3** | Implement network policy: app can only talk to DB on port 3306. Block all other ports between them. |

---

# T3.1 — Build a 3-Tier Docker Architecture

Create a three-tier application architecture:

```text
                   Client
                     │
                     ▼
              ┌─────────────┐
              │    Nginx    │
              │  Frontend   │
              └──────┬──────┘
                     │
              Frontend Network
                     │
                     ▼
              ┌─────────────┐
              │ Flask App   │
              │   Backend   │
              └──────┬──────┘
                     │
              Backend Network
                     │
                     ▼
              ┌─────────────┐
              │    Redis    │
              │    Cache    │
              └─────────────┘
```

The three components are:

### Frontend

```text
Nginx
```

### Backend

```text
Flask application
```

### Cache

```text
Redis
```

Create separate network segments so that each tier has only the connectivity it requires.

A logical segmentation could be:

```text
frontend-net
    │
    ├── nginx
    └── app

backend-net
    │
    ├── app
    └── redis
```

This means the Flask application acts as the connection point between the frontend and backend segments.

---

# T3.2 — Create the 3-Tier Setup with Docker Compose

Create:

```text
docker-compose.yml
```

Define the following services:

```text
nginx
flask
redis
```

Define named networks for the different application segments.

A conceptual structure is:

```yaml
services:
  nginx:
    # Frontend

  app:
    # Backend

  redis:
    # Cache

networks:
  frontend-net:
  backend-net:
```

The intended communication should be:

```text
Nginx
  │
  ▼
Flask App
  │
  ▼
Redis
```

Start the environment:

```bash
docker compose up -d
```

Verify the services:

```bash
docker compose ps
```

Inspect the networks:

```bash
docker network ls
```

Verify communication between the services using their **Compose service names**.

For example, the Flask application should be able to resolve the Redis service using:

```text
redis
```

> 💡 **Key Concept:** Docker Compose provides built-in service discovery. Services on the same Compose network can normally communicate using their service names.

---

# T3.3 — Implement Network Policy

Implement a network policy where:

```text
App → DB : TCP 3306 only
```

All other ports between the application and database should be blocked.

The desired behavior is:

```text
                    Backend Network

                  ┌──────────────┐
                  │  Flask App   │
                  └───────┬──────┘
                          │
                    TCP 3306 ✓
                          │
                          ▼
                  ┌──────────────┐
                  │      DB      │
                  └──────────────┘

Other Ports ────────────────✕
```

Test the required port:

```bash
nc -zv db 3306
```

Then test other ports where applicable and verify that they are not accessible.

> ⚠️ **Important:** A Docker bridge network does not provide arbitrary per-port allow/deny policies by itself. For a true "only TCP 3306 is allowed" policy, use an appropriate firewall/network-control mechanism or a network architecture that exposes only the required database port. Do not assume that simply placing containers on the same network restricts them to a specific port.

---

# 🧠 Docker DNS Resolution

Custom Docker networks provide embedded DNS resolution.

For example:

```text
Container: app
Container: db
Network: devops-net
```

The application can use:

```text
db
```

instead of:

```text
172.x.x.x
```

The logical flow is:

```text
app
 │
 │ Query: db
 ▼
Docker Embedded DNS
 │
 │ Resolves container/service name
 ▼
db → Container IP
```

> 💡 **Best Practice:** Avoid hard-coding dynamically assigned container IP addresses. Use container or service names for application communication.

---

# 🔐 Network Segmentation Best Practices

Use network segmentation to limit unnecessary communication between application components.

A typical architecture might look like:

```text
Internet
   │
   ▼
Nginx / Reverse Proxy
   │
   ▼
Application Network
   │
   ▼
Backend / Data Network
   │
   ▼
Database / Cache
```

Follow the **least-connectivity principle**:

* Frontend should communicate only with required backend services.
* Backend should communicate only with required data services.
* Database services should not be exposed publicly.
* Avoid connecting every container to one shared network.
* Use meaningful network names.
* Avoid hard-coding container IP addresses.
* Expose only required ports.
* Use authentication even when network segmentation is present.

---

# 🧪 Troubleshooting Commands

## List Networks

```bash
docker network ls
```

## Inspect a Network

```bash
docker network inspect <network-name>
```

## Check Container Network Configuration

```bash
docker inspect <container-name>
```

## Test DNS Resolution

From a container:

```bash
ping <container-name>
```

or, when available:

```bash
getent hosts <container-name>
```

## Test HTTP Connectivity

```bash
curl http://<container-name>
```

## Test a Specific TCP Port

```bash
nc -zv <container-name> <port>
```

---

# 🧹 Lab Cleanup

Review running containers:

```bash
docker ps
```

Stop the lab containers:

```bash
docker stop <container-name>
```

Remove containers:

```bash
docker rm <container-name>
```

Review networks:

```bash
docker network ls
```

Remove unused lab networks:

```bash
docker network rm lab-net
docker network rm frontend-net
docker network rm backend-net
```

For the Docker Compose challenge:

```bash
docker compose down
```

> ⚠️ **Caution:** Do not remove networks or containers that are being used by other applications or labs.

---

# 💡 Best-Practice Tips

* Use **custom bridge networks** instead of relying solely on the default bridge.
* Use container/service names for DNS-based service discovery.
* Avoid hard-coding container IP addresses.
* Separate frontend and backend workloads into different networks when appropriate.
* Keep databases on private backend networks.
* Expose only the ports that applications actually require.
* Avoid using host networking unless there is a specific requirement.
* Use Docker Compose named networks for multi-container applications.
* Validate network isolation with actual connectivity tests.
* Use authentication and authorization in addition to network segmentation.
* Treat network segmentation as one layer of a broader defense-in-depth strategy.

---

# ✅ Lab 06 Completion Checklist

* [ ] Understand Docker bridge networking.
* [ ] List Docker networks using `docker network ls`.
* [ ] Create a custom `devops-net` network.
* [ ] Inspect network configuration.
* [ ] Run multiple containers on the same network.
* [ ] Understand Docker embedded DNS.
* [ ] Resolve containers by name.
* [ ] Connect an existing container to a network.
* [ ] Disconnect a container from a network.
* [ ] Create `lab-net`.
* [ ] Run Nginx and a curl container on `lab-net`.
* [ ] Verify container-to-container DNS resolution.
* [ ] Identify container IP addresses.
* [ ] Create separate frontend and backend networks.
* [ ] Connect the application to both networks.
* [ ] Connect the database only to the backend network.
* [ ] Verify network isolation.
* [ ] Understand Docker host networking.
* [ ] Build a 3-tier Nginx → Flask → Redis architecture.
* [ ] Create a Docker Compose configuration with named networks.
* [ ] Verify service-to-service communication.
* [ ] Implement and validate port-specific network controls.
