# Harbor Container Registry Lab

A self-hosted container image registry built with **Harbor, Docker Compose, HTTPS/TLS, and Docker Desktop on WSL2**.

This project demonstrates how to deploy a private container registry, configure TLS trust, manage container images, and verify secure image push and pull operations in a local lab environment.

## Project Overview

Harbor is an open-source container registry that stores, manages, and distributes container images. It provides a web interface, project-based image organization, authentication, and registry APIs.

This lab runs Harbor on Ultramarine Linux under WSL2, using Docker Desktop as the container runtime.

### Key Features

- Private container image registry
- HTTPS access using a local TLS certificate
- Local hostname resolution using `harbor.local`
- Docker client authentication
- Image push, pull, and verification
- Harbor management through Docker Compose
- Persistent registry data for a local development lab

## Architecture

```text
Windows 11
└── WSL2
    └── Ultramarine Linux
        └── Harbor Lab
            ├── NGINX / HTTPS :443
            ├── Harbor Portal
            ├── Harbor Core
            ├── Registry
            ├── Registry Controller
            ├── Harbor Jobservice
            ├── Harbor Database
            ├── Redis
            └── Harbor Log

Docker Desktop
└── Docker Engine
    └── Harbor Containers
```

### Request Flow

```text
Docker CLI / Web Browser
          |
          | HTTPS:443
          v
     harbor.local
          |
          v
    Harbor NGINX
          |
          v
     Harbor Core
          |
          v
    Docker Registry
          |
          v
    Stored Images
```

## Technology Stack

| Technology | Purpose |
|---|---|
| Harbor v2.15.4 | Private container registry |
| Docker Desktop | Container runtime |
| Docker Compose | Service orchestration |
| Ultramarine Linux | WSL2 Linux environment |
| NGINX | HTTPS endpoint and reverse proxy |
| TLS certificates | Encrypted connections and trust |
| Docker CLI | Registry login, push, and pull |

## Project Structure

The current lab uses:

```text
~/harbor-lab/
├── certs/
│   └── ca.crt
└── harbor/
    ├── docker-compose.yml
    ├── harbor.yml
    └── generated Harbor configuration and related files
```

The generated configuration files may vary depending on how Harbor was installed.

**Important:** Keep the certificate private key, credentials, and other secrets out of public repositories.

## Prerequisites

- Windows 11
- WSL2
- Ultramarine Linux
- Docker Desktop with Docker Engine running
- Docker Compose
- A local hostname configured as `harbor.local`
- TLS certificate authority certificate for the local registry

## Configuration

### Registry hostname

```text
harbor.local
```

The hostname is mapped to the local machine so that clients can use a consistent registry address.

For the WSL Linux environment, the hosts entry is:

```text
127.0.0.1 harbor.local
```

The Windows hosts file is also configured for local access:

```text
C:\Windows\System32\drivers\etc\hosts
```

Add the following line if it is not already present:

```text
127.0.0.1 harbor.local
```

The mapping should be appropriate for this local-only lab. Do not assume that `127.0.0.1` is the correct address for clients on other machines.

### HTTPS and TLS

The lab uses:

```text
https://harbor.local
```

The local CA certificate is stored at:

```text
~/harbor-lab/certs/ca.crt
```

The Docker client trust directory on Windows is configured as:

```text
%USERPROFILE%\.docker\certs.d\harbor.local\ca.crt
```

The CA certificate establishes trust in the registry's TLS certificate. Never distribute the CA private key or the registry's private key.

### Published ports

The current Docker Compose configuration binds the registry ports to loopback:

| Host port | Container port | Purpose |
|---:|---:|---|
| 80 | 8080 | HTTP endpoint |
| 443 | 8443 | HTTPS endpoint |
| 1514 | 10514 | Harbor log service |

The configured host bindings are:

```text
127.0.0.1:80->8080
127.0.0.1:443->8443
127.0.0.1:1514->10514
```

These mappings are intended for local access.

## Start and Stop Harbor

### Start Harbor

After starting Docker Desktop, open the Ultramarine terminal and run:

```bash
cd ~/harbor-lab/harbor
sudo docker compose up -d
```

### Check container status

```bash
cd ~/harbor-lab/harbor
sudo docker compose ps
```

The Harbor services should report healthy or running states as appropriate.

### View logs

```bash
sudo docker compose logs --tail=50
```

View logs for an individual service:

```bash
sudo docker compose logs --tail=100 core
```

### Stop Harbor

```bash
cd ~/harbor-lab/harbor
sudo docker compose down
```

This stops and removes the Compose containers and network. Named volumes are retained by default, but verify how persistent data is configured before making changes.

**Avoid** `docker compose down -v` unless you intentionally want to remove the associated named volumes and understand the data-loss implications.

### After a laptop restart

1. Start Docker Desktop.
2. Open Ultramarine Linux.
3. Start Harbor:

```bash
cd ~/harbor-lab/harbor
sudo docker compose up -d
```

4. Verify the services:

```bash
sudo docker compose ps
```

## Access the Harbor Web Portal

Open:

[https://harbor.local](https://harbor.local)

The browser should trust the local TLS certificate. If it shows a certificate warning, verify that the certificate chain and hostname match the certificate being served.

The administrator username is commonly `admin`, but use the password configured during your installation. Do not commit it to this repository.

## Verify the HTTPS Endpoint

From Ultramarine Linux, run:

```bash
curl --cacert ~/harbor-lab/certs/ca.crt \
  -i https://harbor.local/v2/
```

A response such as:

```http
HTTP/1.1 401 Unauthorized
```

is expected when the registry API requires authentication. It confirms that the HTTPS endpoint is reachable and responding; it does not indicate a failed TLS connection.

## Authenticate with Docker

Log in to Harbor:

```bash
docker login harbor.local -u admin
```

Enter the configured Harbor administrator password when prompted.

If Docker reports a certificate error, confirm that the correct CA certificate is installed in the Docker client's trust directory and that the certificate's hostname matches `harbor.local`.

## Push and Pull a Test Image

### 1. Create a Harbor project

In the Harbor portal:

1. Sign in.
2. Create a project named `lab`.
3. Use the project to store your test image.

### 2. Pull a small test image

```bash
docker pull alpine:latest
```

### 3. Tag the image for Harbor

```bash
docker tag alpine:latest harbor.local/lab/alpine:latest
```

### 4. Push the image

```bash
docker push harbor.local/lab/alpine:latest
```

### 5. Verify the image

Check the image locally:

```bash
docker images | grep harbor.local
```

You can also verify the repository and tag in the Harbor web portal under the `lab` project.

### 6. Test a pull

Remove the local tagged copy and pull it again:

```bash
docker image rm harbor.local/lab/alpine:latest
docker pull harbor.local/lab/alpine:latest
```

This verifies that the image can be retrieved from the private registry.

## Troubleshooting

### Harbor services are not running

```bash
cd ~/harbor-lab/harbor
sudo docker compose ps -a
sudo docker compose logs --tail=100
```

Check Docker Desktop and the Docker Engine before troubleshooting individual Harbor components.

### The browser cannot resolve `harbor.local`

Verify the hosts file entries on Windows and Ultramarine Linux. Then test:

```bash
getent hosts harbor.local
```

Expected local result:

```text
127.0.0.1 harbor.local
```

### TLS certificate verification fails

Test with the configured CA:

```bash
curl --cacert ~/harbor-lab/certs/ca.crt \
  -i https://harbor.local/v2/
```

If this works but Docker login fails, verify Docker's own CA trust configuration. The browser and Docker client may use different trust stores.

### Docker login fails

```bash
docker login harbor.local -u admin
```

Confirm the project exists, the account credentials are correct, the HTTPS endpoint is reachable, and Docker trusts the registry's CA.

Do not disable TLS verification as a permanent workaround.

### Port conflicts

Inspect published ports:

```bash
sudo docker compose ps
```

Check whether the HTTPS host port is listening:

```bash
ss -lntp | grep ':443'
```

Resolve port conflicts before changing Harbor's published-port configuration.

## Security Considerations

- This is a local learning environment, not a production registry deployment.
- The registry's published ports are bound to loopback for local access.
- Use TLS and maintain correct certificate trust.
- Never commit passwords, tokens, private keys, or sensitive configuration files.
- Avoid exposing the Harbor administrator account to untrusted users.
- Consider using separate projects and least-privilege accounts for different workloads.
- Back up registry data and Harbor configuration before upgrades or destructive maintenance.
- Review image vulnerability scanning, access control, retention policies, and audit logging before adopting Harbor for production.

## Skills Demonstrated

- Docker and Docker Compose administration
- Private container registry deployment
- Harbor architecture and service troubleshooting
- TLS certificates and CA trust
- DNS/hosts-file configuration
- Docker registry authentication
- Container image tagging, pushing, and pulling
- Linux service and network troubleshooting
- Persistent container data management

## Future Improvements

- [ ] Configure project-level access control and dedicated robot accounts.
- [ ] Explore image vulnerability scanning.
- [ ] Add image retention policies.
- [ ] Automate registry health checks.
- [ ] Document and test registry backup and recovery.
- [ ] Integrate Harbor with CI/CD pipelines.
- [ ] Configure monitoring with Prometheus and Grafana.
- [ ] Explore secure remote access in a separate environment.

## License

This is a personal learning project. Add an appropriate license if you intend to distribute the project publicly.
