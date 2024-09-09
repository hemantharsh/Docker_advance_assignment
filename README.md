Here is the updated `README.md` with all instances of `my_python_image` replaced with `docker_multistage`:

---

# Docker Advanced Assignment

This document provides an overview of the tasks performed during the Docker assignment. The assignment focused on optimizing Dockerfiles, managing Docker networks and volumes, and implementing security best practices.

## 1. Optimize Dockerfile for Build Speed and Image Size

### **Original Dockerfile**

The original Dockerfile was optimized to improve build speed and reduce image size by using multi-stage builds. Here is the original Dockerfile:

```Dockerfile
FROM alpine:3.19

# ensure local python is preferred over distribution python
ENV PATH /usr/local/bin:$PATH

# cannot remove LANG even though https://bugs.python.org/issue19846 is fixed
# last attempted removal of LANG broke many users:
# https://github.com/docker-library/python/pull/570
ENV LANG C.UTF-8

# runtime dependencies
RUN set -eux; \
    apk add --no-cache \
        ca-certificates \
        tzdata \
    ;

ENV GPG_KEY A035C8C19219BA821ECEA86B64E628F8D684696D
ENV PYTHON_VERSION 3.11.10

RUN set -eux; \
    apk add --no-cache --virtual .build-deps \
        gnupg \
        tar \
        xz \
        bluez-dev \
        bzip2-dev \
        dpkg-dev dpkg \
        expat-dev \
        findutils \
        gcc \
        gdbm-dev \
        libc-dev \
        libffi-dev \
        libnsl-dev \
        libtirpc-dev \
        linux-headers \
        make \
        ncurses-dev \
        openssl-dev \
        pax-utils \
        readline-dev \
        sqlite-dev \
        tcl-dev \
        tk \
        tk-dev \
        util-linux-dev \
        xz-dev \
        zlib-dev \
    ; \
    wget -O python.tar.xz "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz"; \
    wget -O python.tar.xz.asc "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz.asc"; \
    GNUPGHOME="$(mktemp -d)"; export GNUPGHOME; \
    gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys "$GPG_KEY"; \
    gpg --batch --verify python.tar.xz.asc python.tar.xz; \
    gpgconf --kill all; \
    rm -rf "$GNUPGHOME" python.tar.xz.asc; \
    mkdir -p /usr/src/python; \
    tar --extract --directory /usr/src/python --strip-components=1 --file python.tar.xz; \
    rm python.tar.xz; \
    cd /usr/src/python; \
    gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)"; \
    ./configure \
        --build="$gnuArch" \
        --enable-loadable-sqlite-extensions \
        $(test "$gnuArch" != 'riscv64-linux-musl' && echo '--enable-optimizations') \
        --enable-option-checking=fatal \
        --enable-shared \
        --with-lto \
        --with-system-expat \
        --without-ensurepip \
    ; \
    nproc="$(nproc)"; \
    EXTRA_CFLAGS="-DTHREAD_STACK_SIZE=0x100000"; \
    LDFLAGS="${LDFLAGS:--Wl},--strip-all"; \
    make -j "$nproc" \
        "EXTRA_CFLAGS=${EXTRA_CFLAGS:-}" \
        "LDFLAGS=${LDFLAGS:-}" \
        "PROFILE_TASK=${PROFILE_TASK:-}" \
    ; \
    rm python; \
    make -j "$nproc" \
        "EXTRA_CFLAGS=${EXTRA_CFLAGS:-}" \
        "LDFLAGS=${LDFLAGS:--Wl},-rpath='\$\$ORIGIN/../lib'" \
        "PROFILE_TASK=${PROFILE_TASK:-}" \
        python \
    ; \
    make install; \
    cd /; \
    rm -rf /usr/src/python; \
    find /usr/local -depth \
        \( \
            \( -type d -a \( -name test -o -name tests -o -name idle_test \) \) \
            -o \( -type f -a \( -name '*.pyc' -o -name '*.pyo' -o -name 'libpython*.a' \) \) \
        \) -exec rm -rf '{}' + \
    ; \
    find /usr/local -type f -executable -not \( -name '*tkinter*' \) -exec scanelf --needed --nobanner --format '%n#p' '{}' ';' \
        | tr ',' '\n' \
        | sort -u \
        | awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
        | xargs -rt apk add --no-network --virtual .python-rundeps \
    ; \
    apk del --no-network .build-deps; \
    python3 --version

# make some useful symlinks that are expected to exist ("/usr/local/bin/python" and friends)
RUN set -eux; \
    for src in idle3 pydoc3 python3 python3-config; do \
        dst="$(echo "$src" | tr -d 3)"; \
        [ -s "/usr/local/bin/$src" ]; \
        [ ! -e "/usr/local/bin/$dst" ]; \
        ln -svT "$src" "/usr/local/bin/$dst"; \
    done

# if this is called "PIP_VERSION", pip explodes with "ValueError: invalid truth value '<VERSION>'"
ENV PYTHON_PIP_VERSION 24.0
ENV PYTHON_SETUPTOOLS_VERSION 65.5.1
ENV PYTHON_GET_PIP_URL https://github.com/pypa/get-pip/raw/def4aec84b261b939137dd1c69eff0aabb4a7bf4/public/get-pip.py
ENV PYTHON_GET_PIP_SHA256 bc37786ec99618416cc0a0ca32833da447f4d91ab51d2c138dd15b7af21e8e9a

RUN set -eux; \
    wget -O get-pip.py "$PYTHON_GET_PIP_URL"; \
    echo "$PYTHON_GET_PIP_SHA256 *get-pip.py" | sha256sum -c -; \
    export PYTHONDONTWRITEBYTECODE=1; \
    python get-pip.py \
        --disable-pip-version-check \
        --no-cache-dir \
        --no-compile \
        "pip==$PYTHON_PIP_VERSION" \
        "setuptools==$PYTHON_SETUPTOOLS_VERSION" \
        wheel \
    ; \
    rm -f get-pip.py; \
    pip --version

CMD ["python3"]
```

### **Optimized Dockerfile**

The Dockerfile was optimized by using multi-stage builds and reducing the final image size. Here is the optimized Dockerfile:

```Dockerfile
# Stage 1: Build Python
FROM alpine:3.19 AS build-stage

ENV PYTHON_VERSION=3.11.10

RUN set -eux; \
    apk add --no-cache --virtual .build-deps \
        gnupg \
        tar \
        xz \
        gcc \
        libc-dev \
        make \
        openssl-dev \
        zlib-dev \
        bzip2-dev \
        readline-dev \
        sqlite-dev \
        build-base; \
    wget -O python.tar.xz "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz"; \
    tar -xvf python.tar.xz; \
    cd Python-$PYTHON_VERSION; \
    ./configure --prefix=/usr --enable-optimizations --with-lto; \
    make -j"$(nproc)"; \
    make install; \
    cd ..; \
    rm -rf Python-$PYTHON_VERSION python.tar.xz; \
    apk del .build-deps

# Add a non-root user
RUN adduser -D myuser

# Stage 2: Final Image with Python Installed
FROM alpine:3.19

# Add a non-root user
RUN adduser -D myuser

# Switch to non-root user
USER myuser

# Copy Python from build-stage
COPY --from=build-stage /usr/local /usr/local

# Define the default command or entrypoint if needed
# CMD ["your_command"]
CMD ["sleep", "3600"]
```

## 2. Set Up a Custom Docker Network

### **Docker Compose File: `docker-compose.yml`**

This file defines two containers connected to a custom Docker network.

```yaml
version: '3.8'

services:
  python-container1:
    image: docker_multistage
    container_name: python-container1
    networks:
      - my_custom_network
    restart: unless-stopped

  python-container2:
    image: docker_multistage
    container_name: python-container2
    networks:
      - my_custom_network
    restart: unless-stopped

networks:
  my_custom_network:
    driver: bridge
```

### **Steps to Use Docker Compose**

1. **Create the Docker Compose File**:
    ```bash
    touch docker-compose

.yml
    ```
   Paste the YAML content into this file.

2. **Start the Docker Daemon**:
    ```bash
    sudo dockerd
    ```

3. **Build the Docker Image**:
    ```bash
    docker buildx build -f Dockerfile_multistage -t docker_multistage .
    ```

4. **Run Docker Compose**:
    ```bash
    docker-compose up -d
    ```

5. **Verify Running Containers**:
    ```bash
    docker-compose ps
    ```

6. **Inspect the Network**:
    ```bash
    docker network inspect my_custom_network
    ```

## 3. Create and Manage Docker Volumes

### **Creating Docker Volumes**:
```bash
docker volume create my_volume
```

### **Using Named Volumes in Containers**:
```bash
docker run -d --name my_container -v my_volume:/data docker_multistage
```

### **List Docker Volumes**:
```bash
docker volume ls
```

### **Inspect Docker Volumes**:
```bash
docker volume inspect my_volume
```

### **Remove Docker Volumes**:
```bash
docker volume rm my_volume
```

### **Using Bind Mounts**:
```bash
docker run -d --name my_container -v /host/path:/container/path docker_multistage
```

### **Example with Multiple Containers**:
```bash
docker run -d --name app1 -v /host/data:/shared_data docker_multistage
docker run -d --name app2 -v /host/data:/shared_data docker_multistage
```

### **Backup and Restore Data from Docker Volumes**

#### **Backing Up Data**:
```bash
docker run --rm -v my_volume:/data -v $(pwd):/backup alpine tar czf /backup/backup.tar.gz -C /data .
```

#### **Restoring Data**:
```bash
docker run --rm -v my_volume:/data -v $(pwd):/backup alpine sh -c "tar xzf /backup/backup.tar.gz -C /data"
```

## 4. Implement Security Best Practices

### **User Permissions**:
```Dockerfile
# Add this to your Dockerfile
RUN adduser -D myuser
USER myuser
```

### **Image Vulnerability Scanning**:
```bash
docker scan docker_multistage
```

### **Secret Management**:
```bash
echo "my_secret" | docker secret create my_secret -
docker service create --name my_service --secret my_secret docker_multistage
```

## 5. Run Containers with Least Privilege

### **Run Containers with a Non-Root User**:
```bash
docker run --user 1000:1000 docker_multistage
```

### **Drop Unnecessary Capabilities**:
```bash
docker run --cap-drop=ALL docker_multistage
```

## 6. Use Docker Bench for Security

### **Install Docker Bench for Security**:
```bash
git clone https://github.com/docker/docker-bench-security.git
cd docker-bench-security
```

### **Run Docker Bench for Security**:
```bash
sudo ./docker-bench-security.sh
```

![Docker Bench for Security](https://github.com/user-attachments/assets/05b009a5-eb7d-40b1-845c-13f875a3b766)
![Docker Bench for Security](https://github.com/user-attachments/assets/62daa2cd-25f3-43bf-b4fc-4de5483a71a4)
![Docker Bench for Security](https://github.com/user-attachments/assets/dc65afef-a243-404a-a8ba-a3935958b2f0)
![Docker Bench for Security](https://github.com/user-attachments/assets/0088c113-3b1d-4a8e-bf17-fb6e53af4fb5)

---

The assignment has been completed.

**THANK YOU!!**
