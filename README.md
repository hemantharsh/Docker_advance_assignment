# Docker_advance_assignment
Docker Assignment Documentation
This document details the steps and commands used to complete the Docker assignment. The assignment involved optimizing a Dockerfile, managing Docker networks and volumes, and implementing security best practices.

1. Optimize Dockerfile for Build Speed and Image Size
Original Dockerfile
The existing Dockerfile was optimized to improve build speed and reduce image size by using multi-stage builds.
orignal Docker file :-
"
#
# NOTE: THIS DOCKERFILE IS GENERATED VIA "apply-templates.sh"
#
# PLEASE DO NOT EDIT IT DIRECTLY.
#

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
	\
	apk add --no-cache --virtual .build-deps \
		gnupg \
		tar \
		xz \
		\
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
	\
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
	\
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
# set thread stack size to 1MB so we don't segfault before we hit sys.getrecursionlimit()
# https://github.com/alpinelinux/aports/commit/2026e1259422d4e0cf92391ca2d3844356c649d0
	EXTRA_CFLAGS="-DTHREAD_STACK_SIZE=0x100000"; \
	LDFLAGS="${LDFLAGS:--Wl},--strip-all"; \
	make -j "$nproc" \
		"EXTRA_CFLAGS=${EXTRA_CFLAGS:-}" \
		"LDFLAGS=${LDFLAGS:-}" \
		"PROFILE_TASK=${PROFILE_TASK:-}" \
	; \
# https://github.com/docker-library/python/issues/784
# prevent accidental usage of a system installed libpython of the same version
	rm python; \
	make -j "$nproc" \
		"EXTRA_CFLAGS=${EXTRA_CFLAGS:-}" \
		"LDFLAGS=${LDFLAGS:--Wl},-rpath='\$\$ORIGIN/../lib'" \
		"PROFILE_TASK=${PROFILE_TASK:-}" \
		python \
	; \
	make install; \
	\
	cd /; \
	rm -rf /usr/src/python; \
	\
	find /usr/local -depth \
		\( \
			\( -type d -a \( -name test -o -name tests -o -name idle_test \) \) \
			-o \( -type f -a \( -name '*.pyc' -o -name '*.pyo' -o -name 'libpython*.a' \) \) \
		\) -exec rm -rf '{}' + \
	; \
	\
	find /usr/local -type f -executable -not \( -name '*tkinter*' \) -exec scanelf --needed --nobanner --format '%n#p' '{}' ';' \
		| tr ',' '\n' \
		| sort -u \
		| awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
		| xargs -rt apk add --no-network --virtual .python-rundeps \
	; \
	apk del --no-network .build-deps; \
	\
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
# https://github.com/docker-library/python/issues/365
ENV PYTHON_SETUPTOOLS_VERSION 65.5.1
# https://github.com/pypa/get-pip
ENV PYTHON_GET_PIP_URL https://github.com/pypa/get-pip/raw/def4aec84b261b939137dd1c69eff0aabb4a7bf4/public/get-pip.py
ENV PYTHON_GET_PIP_SHA256 bc37786ec99618416cc0a0ca32833da447f4d91ab51d2c138dd15b7af21e8e9a

RUN set -eux; \
	\
	wget -O get-pip.py "$PYTHON_GET_PIP_URL"; \
	echo "$PYTHON_GET_PIP_SHA256 *get-pip.py" | sha256sum -c -; \
	\
	export PYTHONDONTWRITEBYTECODE=1; \
	\
	python get-pip.py \
		--disable-pip-version-check \
		--no-cache-dir \
		--no-compile \
		"pip==$PYTHON_PIP_VERSION" \
		"setuptools==$PYTHON_SETUPTOOLS_VERSION" \
		# get-pip.py installs wheel by default, adding in case get-pip defaults change
		wheel \
	; \
	rm -f get-pip.py; \
	\
	pip --version

CMD ["python3"]

"
optimized docker file :-
"
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
"
3.Set up a Custom Docker Network:
To meet the requirements of your assignment and based on the Dockerfile and task descriptions, here's how you can create a Docker Compose file to automate the setup.

Docker Compose File: docker-compose.yml
This file will define:

Two containers based on the people:multistage image.
A custom Docker network to connect the containers.
Container names and network configuration.
Here's the docker-compose.yml:

yaml
Copy code
version: '3.8'

services:
  python-container1:
    image: people:multistage
    container_name: python-container1
    networks:
      - my_custom_network
    restart: unless-stopped

  python-container2:
    image: people:multistage
    container_name: python-container2
    networks:
      - my_custom_network
    restart: unless-stopped

networks:
  my_custom_network:
    driver: bridge
Explanation of the Docker Compose File:
version: '3.8': Specifies the version of the Docker Compose file format.
services: Defines the services (containers) that Compose will manage.
python-container1: A service (container) running the image people:multistage built from your Dockerfile.
python-container2: A second service running the same image.
networks: Both containers are connected to a custom network called my_custom_network.
restart: unless-stopped: Ensures the containers are restarted unless manually stopped.
networks: Defines the custom network named my_custom_network using the bridge driver.
Steps to Use Docker Compose
Create the docker-compose.yml file: In your project directory (e.g., ~/Docker_assignment_2), create a file named docker-compose.yml:

bash
Copy code
touch docker-compose.yml
nano docker-compose.yml
Paste the YAML content from above into this file.

Start the Docker Daemon: Since systemctl doesn't work on your system, open Terminal 1 and start Docker using:

bash
Copy code
sudo dockerd
Build the Docker Image: If you haven’t already built the image from your Dockerfile, run the following in Terminal 2:

bash
Copy code
docker buildx build -f Dockerfile_multistage -t people:multistage .
Run Docker Compose: In Terminal 2, navigate to the directory containing docker-compose.yml and bring up the services:

bash
Copy code
docker-compose up -d
This command will:

Create the custom network (my_custom_network).
Start two containers (python-container1 and python-container2) using the people:multistage image.
Verify Running Containers: Check that the containers are running and connected to the network:

bash
Copy code
docker-compose ps
Inspect the Network: You can inspect the custom network created by Docker Compose:

bash
Copy code
docker network inspect my_custom_network

 Create and Manage Docker Volumes
Creating Docker Volumes:
To create a Docker volume, use the docker volume create command:

bash
Copy code
docker volume create my_volume
This command creates a volume named my_volume.

Using Named Volumes in Containers:
You can mount this volume in a container with the -v or --mount option:

bash
Copy code
docker run -d --name my_container -v my_volume:/data my_python_image
This mounts the my_volume volume to the /data directory in the container.

List Docker Volumes:
To list all Docker volumes:

bash
Copy code
docker volume ls
Inspect Docker Volumes:
To inspect a specific volume:

bash
Copy code
docker volume inspect my_volume
Remove Docker Volumes:
To remove a Docker volume:

bash
Copy code
docker volume rm my_volume
Note: You must ensure no container is using the volume before removing it.

2. Bind Mounts
Using Bind Mounts:
A bind mount allows you to mount a directory from the host machine into a container. For example:

bash
Copy code
docker run -d --name my_container -v /host/path:/container/path my_python_image
This mounts the /host/path directory on the host to /container/path in the container.

Example with Multiple Containers:
You can use bind mounts in a multi-container setup to share data:

bash
Copy code
docker run -d --name app1 -v /host/data:/shared_data my_python_image
docker run -d --name app2 -v /host/data:/shared_data my_python_image
Both app1 and app2 containers will have access to the /host/data directory via /shared_data.

3. Backup and Restore Data from Docker Volumes
Backing Up Data:
You can back up data from a volume by creating an archive of its contents. First, create a temporary container to access the volume:

bash
Copy code
docker run --rm -v my_volume:/data -v $(pwd):/backup alpine tar czf /backup/backup.tar.gz -C /data .
This command creates a backup file backup.tar.gz in the current directory.

Restoring Data:
To restore data, you can use a similar approach:

bash
Copy code
docker run --rm -v my_volume:/data -v $(pwd):/backup alpine sh -c "tar xzf /backup/backup.tar.gz -C /data"
This command extracts the backup.tar.gz file to the my_volume volume.

4. Implement Security Best Practices
User Permissions:
Create a non-root user in your Dockerfile to avoid running containers as root:

Dockerfile
Copy code
# Add this to your Dockerfile
RUN adduser -D myuser
USER myuser
Image Vulnerability Scanning:
Use tools like trivy or docker scan to scan your Docker images for vulnerabilities:

bash
Copy code
docker scan my_python_image
Secret Management:
Use Docker secrets for managing sensitive data. First, create a secret:

bash
Copy code
echo "my_secret" | docker secret create my_secret -
Use the secret in your Docker service:

bash
Copy code
docker service create --name my_service --secret my_secret my_python_image
5. Run Containers with Least Privilege
Ensure containers are run with the least privilege necessary. Avoid running containers as root:

bash
Copy code
docker run --user 1000:1000 my_python_image
Use Docker’s security options, such as --cap-drop to remove unnecessary capabilities:

bash
Copy code
docker run --cap-drop=ALL my_python_image
6. Use Docker Bench for Security
Install Docker Bench for Security:
Clone the Docker Bench repository:

bash
Copy code
git clone https://github.com/docker/docker-bench-security.git
cd docker-bench-security
Run Docker Bench for Security:

bash
Copy code
sudo ./docker-bench-security.sh
![image](https://github.com/user-attachments/assets/05b009a5-eb7d-40b1-845c-13f875a3b766)
![image](https://github.com/user-attachments/assets/62daa2cd-25f3-43bf-b4fc-4de5483a71a4)
![image](https://github.com/user-attachments/assets/dc65afef-a243-404a-a8ba-a3935958b2f0)
![image](https://github.com/user-attachments/assets/0088c113-3b1d-4a8e-bf17-fb6e53af4fb5)

The Assignment has been completed
THANK YOU!! 


