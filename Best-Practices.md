
##  Security Best Practices

### 1. **Use Minimal Base Images**

* Use images like `alpine`, `debian-slim`, or `distroless` to reduce the attack surface.
* Smaller images have fewer vulnerabilities.

### 2. **Keep Images Updated**

* Regularly update base and custom images to patch known vulnerabilities.
* Automate vulnerability scanning with tools like **Trivy**, **Clair**, or **Docker Scout**.

### 3. **Use a `.dockerignore` File**

* Exclude sensitive files and unnecessary context (e.g., `.git`, `.env`, credentials).

### 4. **Run as Non-Root User**

* Avoid running containers as the `root` user.
* Create and use a dedicated user inside the Dockerfile:

```dockerfile
RUN useradd -m appuser
USER appuser
```

### 5. **Use Read-Only Filesystems**

* Mount containers with read-only file systems if possible:

```bash
docker run --read-only myimage
```

### 6. **Limit Capabilities**

* Drop unnecessary Linux capabilities:

```bash
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myimage
```

### 7. **Use Secrets Management**

* Avoid baking secrets into images.
* Use Docker secrets or external tools like AWS Secrets Manager, HashiCorp Vault.

### 8. **Scan Images for Vulnerabilities**

* Use tools like:

  * `docker scan`
  * `Trivy`
  * `Grype`

### 9. **Enable SELinux/AppArmor**

* Use Linux security modules to restrict container access to host resources.

### 10. **Restrict Resource Usage**

* Limit CPU and memory usage to prevent DoS:

```bash
docker run --memory=256m --cpus=0.5 myimage
```

---

##  Networking Best Practices

### 1. **Use User-Defined Bridge Networks**

* Prefer custom networks over the default bridge.
* Provides DNS-based service discovery by container name.

```bash
docker network create my_custom_network
```

### 2. **Isolate Containers by Network**

* Create separate networks for frontend, backend, and database layers.
* Reduce lateral movement in case of compromise.

### 3. **Avoid Host Network Mode**

* Avoid `--network host` unless absolutely necessary.
* This bypasses Docker's network isolation.

### 4. **Expose Only Required Ports**

* Minimize surface by publishing only essential ports.

```bash
docker run -p 80:80 myimage  # Avoid exposing 0.0.0.0:ALL
```

### 5. **Use Firewalls and Network Policies**

* Implement firewall rules or tools like Cilium/Calico in orchestrated environments (e.g., Kubernetes).

### 6. **Secure Inter-Container Communication**

* Use TLS for internal services if applicable.
* Consider using service mesh tools (e.g., Istio, Linkerd).

### 7. **Logging and Monitoring**

* Enable audit logging for container and network events.
* Integrate with monitoring tools like Prometheus, Grafana, ELK/EFK stack.
