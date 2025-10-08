Perfect ✅ — I’ll generate a **ready-to-use `Docker_Documentation.md`** with all updates (correct ports, Dockerfiles, `.dockerignore`, and instructions).

Then I can also create a **Word (`.docx`) version** for your repo or documentation purposes.

Here’s the **full Markdown content**:

---

# Docker Documentation for AI/ML Microservices

## Project Overview

This project contains three microservices:

1. **API-Gateway** – Routes API requests
2. **AIML-testcase** – AI/ML model testing service
3. **jobtestcase** – Job integration service

All services use multi-stage Docker builds to optimize image size. Redis is used for caching and messaging.

---

## 1. API-Gateway

**Port Mapping:** `8000:8000`

### Dockerfile

```dockerfile
# -------------------------
# 1) BUILDER STAGE
# -------------------------
FROM python:3.11-slim AS builder

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc g++ make default-libmysqlclient-dev pkg-config \
 && rm -rf /var/lib/apt/lists/*

RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH" PIP_NO_CACHE_DIR=1

COPY requirements.txt .
RUN pip install --upgrade pip && pip install --prefer-binary -r requirements.txt

# -------------------------
# 2) RUNTIME STAGE
# -------------------------
FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    libmariadb3 \
 && rm -rf /var/lib/apt/lists/*

COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH" PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1

COPY . .

EXPOSE 8000

CMD ["python", "gateway.py"]
```

### .dockerignore

```plaintext
.git
.gitignore
__pycache__/
*.pyc
*.log
venv/
myenv/
.env
tests/
Jenkinsfile
sonar-project.properties
*.md
Dockerfile
.dockerignore
```

### Build & Run

```bash
docker build -t api-gateway:latest ./API-Gateway
docker run -d -p 8000:8000 --name api-gateway api-gateway:latest
```

---

## 2. AIML-testcase

**Port Mapping:** `8002:8002`

**Redis Port:** `6379`

### Dockerfile

```dockerfile
FROM python:3.11-slim AS python-builder

WORKDIR /app
COPY requirements.txt .

RUN sed -i 's/networkx==3.5/networkx==3.4.2/' requirements.txt \
    && python -m pip install --upgrade pip \
    && python -m pip install --prefix=/install --no-cache-dir -r requirements.txt

FROM nvidia/cuda:12.1.1-runtime-ubuntu22.04 AS final

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 python3-venv python3-pip \
    default-mysql-client \
    redis-server \
    ffmpeg \
    curl \
 && apt-get clean && rm -rf /var/lib/apt/lists/*

COPY --from=python-builder /install/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
COPY --from=python-builder /install/bin /usr/local/bin

ENV PYTHONPATH=/usr/local/lib/python3.11/site-packages:$PYTHONPATH \
    PATH=/usr/local/bin:$PATH \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

RUN python -m spacy download en_core_web_md

COPY . .
RUN chmod +x start.sh

EXPOSE 8002 6379

CMD ["bash", "-c", "redis-server --daemonize yes && echo 'Redis started...' && python Integration.py"]
```

### .dockerignore

```plaintext
.git
.gitignore
__pycache__/
*.pyc
*.pyo
*.log
venv/
env/
myenv/
tests/
.coverage
pytest_cache/
htmlcov/
.env
Dockerfile
.dockerignore
Jenkinsfile
sonar-project.properties
.vscode/
.idea/
*.swp
*.bak
*.md
Readme.md
```

### Build & Run

```bash
docker build -t aiml-testcase:latest ./AIML-testcase
docker run -d -p 8002:8002 -p 6379:6379 --gpus all --name aiml-testcase aiml-testcase:latest
```

---

## 3. jobtestcase

**Port Mapping:** `8003:8003`

**Redis Port:** `6380`

### Dockerfile

```dockerfile
FROM python:3.11-slim AS python-builder

WORKDIR /app
COPY requirements.txt .

RUN sed -i 's/networkx==3.5/networkx==3.4.2/' requirements.txt \
    && python -m pip install --upgrade pip \
    && python -m pip install --prefix=/install --no-cache-dir -r requirements.txt

FROM nvidia/cuda:12.1.1-runtime-ubuntu22.04 AS final

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 python3-venv python3-pip \
    default-mysql-client \
    redis-server \
    ffmpeg \
    curl \
 && apt-get clean && rm -rf /var/lib/apt/lists/*

COPY --from=python-builder /install/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
COPY --from=python-builder /install/bin /usr/local/bin

ENV PYTHONPATH=/usr/local/lib/python3.11/site-packages:$PYTHONPATH \
    PATH=/usr/local/bin:$PATH \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

RUN python -m spacy download en_core_web_md

COPY . .
RUN chmod +x start.sh || true

EXPOSE 8003 6380

CMD ["bash", "-c", "redis-server --daemonize yes && echo 'Redis started...' && python integration.py"]
```

### .dockerignore

```plaintext
.git
.gitignore
__pycache__/
*.pyc
*.log
venv/
env/
myenv/
tests/
.coverage
pytest_cache/
htmlcov/
.env
Dockerfile
.dockerignore
Jenkinsfile
sonar-project.properties
.vscode/
.idea/
*.swp
*.bak
*.tmp
README.MD
*.md
.DS_Store
Thumbs.db
```

### Build & Run

```bash
docker build -t jobtestcase:latest ./jobtestcase
docker run -d -p 8003:8003 -p 6380:6379 --gpus all --name jobtestcase jobtestcase:latest
```

---

## Docker Compose (Optional)

```yaml
version: "3.8"

services:
  api-gateway:
    build: ./API-Gateway
    ports:
      - "8000:8000"
    depends_on:
      - aiml-testcase
      - jobtestcase

  aiml-testcase:
    build: ./AIML-testcase
    ports:
      - "8002:8002"
      - "6379:6379"

  jobtestcase:
    build: ./jobtestcase
    ports:
      - "8003:8003"
      - "6380:6379"

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"
```

---

✅ This documentation is now ready to include in your repo.

