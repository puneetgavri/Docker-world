

---

# 🐳 Dockerfile Best Practices & Optimization Guide

## 🧱 1. Use the Right Base Image

* **Start minimal:** Choose a smaller base image like:

  * `alpine` (very small, but may require extra setup)
  * `debian:bookworm-slim` (balance between size & compatibility)
  * `gcr.io/distroless/*` (for production — no shell or package manager)
* **Match the language version:** Use tags like `python:3.11-slim` or `node:20-alpine` to avoid ambiguity.
* **Pin versions:** Always specify exact versions to ensure deterministic builds.

  ```dockerfile
  FROM python:3.11-slim
  ```

---

## ⚙️ 2. Multi-Stage Builds

* **Use multi-stage builds** to separate build dependencies from runtime.
* Example:

  ```dockerfile
  FROM golang:1.22 AS builder
  WORKDIR /app
  COPY . .
  RUN go build -o main .

  FROM gcr.io/distroless/base
  COPY --from=builder /app/main /main
  CMD ["/main"]
  ```
* ✅ Keeps final image small
* ✅ Improves security (no compilers/tools in final image)

---

## 🧹 3. Reduce Image Size

* **Clean up after installing packages:**

  ```dockerfile
  RUN apt-get update && apt-get install -y build-essential \
      && rm -rf /var/lib/apt/lists/*
  ```
* **Combine RUN commands:**

  ```dockerfile
  RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
  ```

  → Minimizes image layers.
* **.dockerignore everything not needed:**

  * Prevents large context uploads during build.

  ```
  node_modules
  .git
  .env
  __pycache__
  ```

---

## 🧩 4. Layering & Caching Strategy

* **Order Dockerfile instructions wisely:**

  * Place **less frequently changing layers first**, such as dependency installation.
  * Place **code COPY** statements near the end.
* Example for Python:

  ```dockerfile
  COPY requirements.txt .
  RUN pip install --no-cache-dir -r requirements.txt
  COPY . .
  ```
* This ensures dependency caching is preserved if only source code changes.

---

## ⚡ 5. Optimize Build Performance

* **Use `--progress=plain --no-cache`** during debugging builds.
* **Leverage BuildKit** (`DOCKER_BUILDKIT=1 docker build .`) for parallel builds and better caching.
* **Use target stages** to build only what you need:

  ```bash
  docker build --target builder .
  ```

---

## 🔒 6. Security Best Practices

* **Run as non-root user:**

  ```dockerfile
  RUN useradd -m appuser
  USER appuser
  ```
* **Avoid secrets in Dockerfile:**

  * Use build-time secrets (`--secret id=mysecret,src=secret.txt`) or environment variables.
  ```dockerfile
  FROM alpine:latest

  # Use the secret during a RUN command; the secret is mounted temporarily
  RUN --mount=type=secret,id=mysecret cat /run/secrets/mysecret > /tmp/secret_output.txt
  ```

* **Use distroless images for production:**
  Removes shell, `apt`, and other unnecessary tools.
* **Regularly scan images:**

  ```bash
  docker scan myimage:latest
  ```

---

## 🧰 7. Efficient Package Installation

* **For Python:**

  ```dockerfile
  RUN pip install --no-cache-dir -r requirements.txt
  ```
* **For Node.js:**

  ```dockerfile
  RUN npm ci --omit=dev
  ```
* **For OS packages:**

  * Use `apt-get install --no-install-recommends`
  * Clean up after install to reduce size.

---

## 🧾 8. Environment & Configuration

* **Set environment variables for reliability:**

  ```dockerfile
  ENV PYTHONDONTWRITEBYTECODE=1
  ENV PYTHONUNBUFFERED=1
  ```
* **Define work directory:**

  ```dockerfile
  WORKDIR /app
  ```
* **Use `CMD` or `ENTRYPOINT` properly:**

  ```dockerfile
  CMD ["python", "app.py"]
  ```

---

## 🔁 9. Logging & Observability

* **Write logs to stdout/stderr** (let Docker handle log collection).
* Avoid writing logs to files inside the container unless explicitly needed.

---

## 🧪 10. Testing & CI/CD Integration

* **Lint your Dockerfile:**

  ```bash
  docker run --rm -i hadolint/hadolint < Dockerfile
  ```
* **Automate image builds & scans** in CI/CD (GitHub Actions, GitLab CI, etc.)
* **Use tagging strategy:**

  * `latest` for stable release
  * `dev`, `test`, `prod`, or `sha-<commit>` for clarity

---

## 📦 11. Runtime Optimization

* **Minimize running processes:** One process per container is best.
* **Use health checks:**

  ```dockerfile
  HEALTHCHECK CMD curl -f http://localhost:8000/health || exit 1
  ```
* **Leverage read-only root filesystem:**

  ```dockerfile
  RUN mkdir -p /app/data
  VOLUME /app/data
  ```

---

## 📉 12. Image Size Verification

* Check final image size:

  ```bash
  docker images | grep myapp
  ```
* Use `dive` to inspect and analyze layers:

  ```bash
  dive myimage:latest
  ```

---
