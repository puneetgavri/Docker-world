
# Docker Security Best Practices  
Detailed Guide: Daemon, Image, Container

Docker helps isolate workloads but must be hardened for true security. These best practices address the three main security areas: **Docker Daemon**, **Image**, and **Container Runtime**.

***

## 1. Docker Daemon Security

### 1.1 **Do Not Expose the Docker Daemon Socket**
- The Docker daemon listens by default only on the local Unix socket, `/var/run/docker.sock`.
- Exposing it over TCP (i.e., via `tcp://`) allows remote execution of Docker commands.
- **Risk:** If exposed to the internet, an attacker could gain control of the host.
- **Best Practice:**  
  - Keep TCP disabled unless strictly needed for remote admin.
  - Restrict access to Unix socket only to trusted users via the `docker` group.
- **Example:**  
  ```bash
  sudo usermod -aG docker youruser  # Add user to docker group
  ```

### 1.2 **Protect TCP Sockets with TLS If You Must Use Them**
- If you must expose Docker’s TCP socket, always use TLS with certificates.
- Without TLS, anyone accessing your host can issue Docker API calls.
- **Best Practice:**  
  ```bash
  dockerd --tlsverify \
    --tlscacert=ca.pem \
    --tlscert=server-cert.pem \
    --tlskey=server-key.pem \
    -H=0.0.0.0:2376
  ```
- Regularly rotate certificates to minimize risk.

### 1.3 **Enable Rootless Docker Where Possible**
- Docker daemon and containers run as `root` by default.
- A compromise at the daemon or container level is a host system risk.
- **Rootless Mode:** Runs Docker as a non-root user. Reduces impact if compromised.
- **Limitations:** Some features (privileged containers, overlay networks) may not be available.
- **Example:**  
  ```bash
  dockerd-rootless.sh
  ```

### 1.4 **Keep Docker and Host OS Up to Date**
- Security patches are regularly released by Docker and Linux distros.
- **Best Practice:**  
  - Regularly update both Docker and the host operating system.
  ```bash
  sudo apt-get update && sudo apt-get upgrade docker-ce
  ```

### 1.5 **Restrict Inter-Container Communication (ICC)**
- By default, Docker allows containers to freely communicate on the bridge network.
- **Risk:** Malicious process/container can attack peers.
- **Best Practice:**  
  - Disable ICC with `--icc=false` on the daemon.
  - Create custom, restrictive networks as needed.
  ```bash
  dockerd --icc=false
  docker network create --internal app-net
  ```

### 1.6 **Enable SELinux, Seccomp, and AppArmor**
- **SELinux/AppArmor:** Enforce policy-based access control for containers.
- **Seccomp:** Restricts dangerous syscalls to mitigate risk.
- **Example:**  
  ```bash
  docker run --security-opt seccomp=default.json myimage
  docker run --security-opt apparmor=docker-default myimage
  ```
- **Tip:** Stick to Docker and OS defaults, or create custom profiles if needed.

### 1.7 **Harden Your Host OS**
- **Defense-in-depth:**  
  - Regularly update kernel  
  - Enable firewalls  
  - Restrict SSH/direct host access  
  - Only necessary admin users should have access
  - Separate Docker hosts from other workloads if possible.

### 1.8 **Enable User Namespace Remapping**
- Maps container’s root user within an isolated namespace range.
- **Prevents:** Container processes from mapping to root on host.
- **Best Practice:**  
  ```bash
  dockerd --userns-remap=default
  ```
- **Note:** Review compatibility if your containers use advanced features (e.g., device drivers).

***

## 2. Docker Image Security

### 2.1 **Use Trusted and Minimal Base Images**
- Avoid untrusted images from random sources.
- Prefer official/verified images (see Docker Hub filters).
- **Minimal images:** Fewer packages means fewer vulnerabilities (e.g., Alpine).
- **Example:**  
  ```dockerfile
  FROM alpine:latest
  ```

### 2.2 **Rebuild Images Regularly**
- Docker images are immutable; updates and patches released post-build won’t be applied.
- **Best Practice:** Automate regular rebuilds; restart containers for the updates to take effect.
- **Example:**  
  Use [Watchtower](https://containrrr.dev/watchtower) or CI/CD pipelines to rebuild and redeploy.

### 2.3 **Scan Images for Vulnerabilities**
- Use scanners: [Docker Scout](https://docs.docker.com/scout/), [Trivy](https://aquasecurity.github.io/trivy/), [Anchore](https://anchore.com/).
- Integrate into build and release processes.
- **Example:**  
  ```bash
  trivy image myorg/myapp:latest
  ```

### 2.4 **Verify Image Authenticity With Docker Content Trust**
- Sign images and verify signatures before deploy.
- Only run images you can verify.
- **Best Practice:**  
  ```bash
  export DOCKER_CONTENT_TRUST=1
  ```

### 2.5 **Lint Dockerfiles Before Building**
- Linting identifies security issues and best practice violations in your Dockerfiles.
- Tool: [Hadolint](https://github.com/hadolint/hadolint)
- **Example:**  
  ```bash
  hadolint Dockerfile
  ```
- Integrate into CI pipelines for automated reviews.

***

## 3. Docker Container Security

### 3.1 **Don’t Expose Unneeded Ports**
- Open ports can be probed and attacked.
- Only expose what's necessary for your service.
- **Dockerfile Example:**  
  ```dockerfile
  EXPOSE 5000
  ```
- **Run Command Example:**  
  ```bash
  docker run -p 5000:5000 myimage
  ```

### 3.2 **Never Use Privileged Mode Unless Absolutely Required**
- Privileged mode (`--privileged`) disables container isolation and gives full access to the host.
- **Example:**  
  ```bash
  docker run --privileged myimage  # Dangerous! Use only if you know what you're doing.
  ```
- Avoid in production whenever possible.

### 3.3 **Drop Unnecessary Linux Capabilities**
- Reduce privileges by dropping all capabilities, then adding only those really needed.
- **Example:**  
  ```bash
  docker run --cap-drop=ALL --cap-add=CHOWN myimage
  ```
- Review what your app needs and don’t grant excessive capabilities.

### 3.4 **Set Resource Limits**
- Prevent individual containers from consuming unlimited CPU or memory.
- Protects from denial-of-service attacks and noisy neighbors.
- **Example:**  
  ```bash
  docker run -m 256m --cpus=2 myimage
  ```

### 3.5 **Run Container Processes as Non-Root User**
- Specify a non-root user (recommended) in your Dockerfile or at runtime.
- **Dockerfile Example:**  
  ```dockerfile
  USER appuser
  ```
- **Runtime Example:**  
  ```bash
  docker run --user=1001 myimage
  ```

### 3.6 **Prevent Privilege Escalation in Containers**
- Block new privileges using `no-new-privileges`.
- Prevents setuid/setgid from making root escalation possible.
- **Example:**  
  ```bash
  docker run --security-opt=no-new-privileges:true myimage
  ```

### 3.7 **Enable Read-Only Filesystem Mode**
- Restrict container writes to pre-designated volume mounts only.
- Prevents attackers from changing binaries or modifying files in the container.
- **Example:**  
  ```bash
  docker run --read-only -v /data myimage
  ```

### 3.8 **Use Dedicated Secrets Management**
- Don’t bake secrets(env vars, certs, API keys) into images or pass as environment variables.
- Use solutions like Docker Secrets (Swarm), HashiCorp Vault, AWS/GCP/K8s secrets.
- **Swarm Example:**  
  ```bash
  echo "mysecret" | docker secret create app_secret -
  docker service create --secret app_secret myimage
  ```

***
