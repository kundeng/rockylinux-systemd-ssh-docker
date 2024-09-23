#  Rocky Linux Container with Systemd and SSH Support

This project provides a Docker container based on the latest Rocky Linux image with systemd and SSH enabled. It includes a preconfigured `vagrant` user and allows you to optionally configure an additional user with certificate login at container runtime. The container also supports options to disable the `vagrant` user and enable SSH key-based authentication for inter-container SSH access.

## Table of Contents

- [Features](#features)
- [Requirements](#requirements)
- [Usage](#usage)
  - [Building the Docker Image](#building-the-docker-image)
  - [Running Containers](#running-containers)
    - [Single Container](#single-container)
    - [Multiple Containers with Docker Compose](#multiple-containers-with-docker-compose)
- [Configuration Options](#configuration-options)
- [Files](#files)
  - [Dockerfile](#dockerfile)
  - [entrypoint.sh](#entrypointsh)
  - [docker-compose.yml](#docker-composeyml)
- [Troubleshooting](#troubleshooting)
- [Security Considerations](#security-considerations)
- [License](#license)

---

## Features

- **Based on Rocky Linux 9:** Provides a modern and stable base image.
- **Systemd Support:** Configured to run systemd within the container.
- **SSH Server Enabled:** Allows SSH access into the container.
- **Preconfigured `vagrant` User:** Includes the default Vagrant user with key-based authentication.
- **Optional Additional User:** Supports creating an extra user at runtime with SSH key-based login.
- **Inter-Container SSH Access:** Containers can SSH into each other using shared SSH keys.
- **Flexible SSH Key Management:** Assumes default locations for SSH keys, making configuration easier.
- **Option to Disable `vagrant` User:** Enhances security by allowing you to disable the default user.

## Requirements

- **Docker Engine:** Version 19.03 or higher.
- **Docker Compose:** Version 1.27 or higher (if using Docker Compose).
- **SSH Key Pair:** RSA or ED25519 key pair for SSH authentication.
- **Host Machine:** Linux-based system is recommended for running systemd inside Docker.

## Usage

### Building the Docker Image

Clone this repository and navigate to the project directory:

```bash
git clone https://github.com/yourusername/rockylinux-systemd-ssh-docker.git
cd rockylinux-systemd-ssh-docker
```

Build the Docker image:

```bash
docker build -t ultimate-rocky-container .
```

### Running Containers

#### Single Container

1. **Prepare SSH Keys**

   Place your SSH private and public keys in the `ssh_keys` directory:

   ```bash
   mkdir ssh_keys
   cp /path/to/your/id_rsa ssh_keys/
   cp /path/to/your/id_rsa.pub ssh_keys/
   ```

   Ensure the keys have the correct permissions:

   ```bash
   chmod 600 ssh_keys/id_rsa
   chmod 644 ssh_keys/id_rsa.pub
   ```

2. **Run the Docker Container**

   ```bash
   docker run -d --name my_container \
       -e EXTRA_USER=extrauser \
       -v $(pwd)/ssh_keys:/ssh_keys:ro \
       -p 2222:22 \
       --privileged \
       -v /sys/fs/cgroup:/sys/fs/cgroup:ro \
       ultimate-rocky-container
   ```

   **Explanation:**

   - `-e EXTRA_USER=extrauser`: Specifies the additional user to create.
   - `-v $(pwd)/ssh_keys:/ssh_keys:ro`: Mounts the SSH keys directory into the container.
   - `-p 2222:22`: Maps port 22 in the container to port 2222 on the host.
   - `--privileged` and `-v /sys/fs/cgroup:/sys/fs/cgroup:ro`: Required for systemd to function properly.

3. **SSH into the Container**

   ```bash
   ssh -p 2222 extrauser@localhost -i ssh_keys/id_rsa
   ```

#### Multiple Containers with Docker Compose

1. **Prepare SSH Keys**

   Place your SSH keys in the `ssh_keys` directory as described above.

2. **Use Docker Compose**

   Create a `docker-compose.yml` file (provided below) and run:

   ```bash
   docker-compose up -d
   ```

3. **Test SSH Between Containers**

   - **Get the IP address of `container2`:**

     ```bash
     docker-compose exec container2 hostname -I
     ```

   - **SSH from `container1` to `container2`:**

     ```bash
     docker-compose exec container1 ssh extrauser@<container2_ip>
     ```

## Configuration Options

- **EXTRA_USER**: (Optional) The username of the additional user to create at runtime.
- **DISABLE_VAGRANT_USER**: (Optional) Set to `yes` to disable the `vagrant` user.
- **SSH keys**: Place your `id_rsa` (private key) and `id_rsa.pub` (public key) in the `ssh_keys` directory.

## Files

### Dockerfile

```dockerfile
# Dockerfile content here (provided below)
```

### entrypoint.sh

```bash
#!/bin/bash

# entrypoint.sh content here (provided below)
```

### docker-compose.yml

```yaml
version: '3'
services:
  container1:
    build: .
    container_name: container1
    privileged: true
    volumes:
      - ./ssh_keys:/ssh_keys:ro
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    environment:
      - EXTRA_USER=extrauser
      - DISABLE_VAGRANT_USER=yes
    ports:
      - "2222:22"

  container2:
    build: .
    container_name: container2
    privileged: true
    volumes:
      - ./ssh_keys:/ssh_keys:ro
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    environment:
      - EXTRA_USER=extrauser
      - DISABLE_VAGRANT_USER=yes
    ports:
      - "2223:22"
```

---

## Dockerfile

```dockerfile
# Base image using Rocky Linux with systemd support
FROM rockylinux:9

# Environment settings
ENV container docker
ENV user=vagrant
ENV ssh_port=22

# Install necessary packages including systemd, SSH server, and sudo
RUN dnf -y update && \
    dnf -y install systemd openssh-server sudo curl && \
    dnf clean all

# Enable systemd and configure it to run properly within a container
RUN ([ -d /lib/systemd/system/sysinit.target.wants ] && \
    cd /lib/systemd/system/sysinit.target.wants/ && \
    for i in *; do [ "$i" != "systemd-tmpfiles-setup.service" ] && rm -f "$i"; done) && \
    rm -f /lib/systemd/system/multi-user.target.wants/* && \
    rm -f /etc/systemd/system/*.wants/* && \
    rm -f /lib/systemd/system/local-fs.target.wants/* && \
    rm -f /lib/systemd/system/sockets.target.wants/*udev* && \
    rm -f /lib/systemd/system/sockets.target.wants/*initctl* && \
    rm -f /lib/systemd/system/basic.target.wants/* && \
    rm -f /lib/systemd/system/anaconda.target.wants/*

# Configure SSH server and set up the vagrant user
RUN systemctl enable sshd.service && \
    useradd "${user}" && \
    mkdir -p /home/"${user}"/.ssh && \
    curl -LSs https://raw.githubusercontent.com/hashicorp/vagrant/master/keys/vagrant.pub > /home/"${user}"/.ssh/authorized_keys && \
    chmod 600 /home/"${user}"/.ssh/authorized_keys && \
    chmod 700 /home/"${user}"/.ssh && \
    chown -R "${user}":"${user}" /home/"${user}"/.ssh && \
    echo "${user} ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers && \
    echo "RSAAuthentication yes" >> /etc/ssh/sshd_config && \
    echo "PubkeyAuthentication yes" >> /etc/ssh/sshd_config && \
    sed -i "s/^#PermitRootLogin yes/PermitRootLogin no/" /etc/ssh/sshd_config && \
    sed -i "s/^#PasswordAuthentication yes/PasswordAuthentication no/" /etc/ssh/sshd_config

# Expose SSH port
EXPOSE ${ssh_port}

# Copy the entrypoint script into the container
COPY entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh

# Create a directory for SSH keys (to be mounted at runtime)
RUN mkdir -p /ssh_keys
VOLUME ["/ssh_keys"]

# Initialize systemd
VOLUME [ "/sys/fs/cgroup" ]

# Set the entrypoint to the custom script
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
CMD ["/usr/sbin/init"]
```

---

## entrypoint.sh
```
#!/bin/bash

# Function to create a user with SSH key
create_user() {
    local username="$1"
    local ssh_pub_key_file="/ssh_keys/id_rsa.pub"

    if id "$username" &>/dev/null; then
        echo "User $username already exists."
    else
        echo "Creating user $username..."
        useradd "$username"
        mkdir -p /home/"$username"/.ssh

        if [ -f "$ssh_pub_key_file" ]; then
            cat "$ssh_pub_key_file" > /home/"$username"/.ssh/authorized_keys
            chmod 600 /home/"$username"/.ssh/authorized_keys
        else
            echo "Public key file $ssh_pub_key_file not found. Skipping SSH key setup for $username."
        fi

        chmod 700 /home/"$username"/.ssh
        chown -R "$username":"$username" /home/"$username"
        echo "$username ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
    fi
}

# Function to set up SSH keys for SSH into other containers
setup_ssh_keys() {
    local username="$1"
    local ssh_private_key_file="/ssh_keys/id_rsa"

    if [ -f "$ssh_private_key_file" ]; then
        echo "Setting up SSH private key for user $username..."
        mkdir -p /home/"$username"/.ssh
        cp "$ssh_private_key_file" /home/"$username"/.ssh/id_rsa
        chmod 600 /home/"$username"/.ssh/id_rsa
        chown "$username":"$username" /home/"$username"/.ssh/id_rsa
    else
        echo "No SSH private key found at $ssh_private_key_file. Skipping SSH key setup for $username."
    fi
}

# Check if EXTRA_USER is set and not empty
if [ -n "$EXTRA_USER" ]; then
    create_user "$EXTRA_USER"
    setup_ssh_keys "$EXTRA_USER"
fi

# Optionally disable vagrant user
if [ "$DISABLE_VAGRANT_USER" = "yes" ]; then
    echo "Disabling vagrant user..."
    usermod -L "$user"
    usermod -s /sbin/nologin "$user"
fi

# Start SSH service
systemctl start sshd

# Execute the original command
exec "$@"

```

---

## docker-compose.yml

```yaml
version: '3'
services:
  container1:
    build: .
    container_name: container1
    privileged: true
    volumes:
      - ./ssh_keys:/ssh_keys:ro
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    environment:
      - EXTRA_USER=extrauser
      - DISABLE_VAGRANT_USER=yes
    ports:
      - "2222:22"

  container2:
    build: .
    container_name: container2
    privileged: true
    volumes:
      - ./ssh_keys:/ssh_keys:ro
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    environment:
      - EXTRA_USER=extrauser
      - DISABLE_VAGRANT_USER=yes
    ports:
      - "2223:22"
```

---

## Troubleshooting

- **SSH Permission Denied:**

  - Ensure the SSH keys have the correct permissions.
  - Verify that the public key exists at `/ssh_keys/id_rsa.pub` inside the container.

- **Systemd Not Working:**

  - Ensure you're using the `--privileged` flag and mounting `/sys/fs/cgroup`.

- **SSH Host Key Verification Failed:**

  - Use `-o StrictHostKeyChecking=no` for testing.
  - Manage the `known_hosts` file appropriately.

## Security Considerations

- **Protect Your Private Keys:**

  - Do not share private keys.
  - Ensure they have appropriate permissions (`chmod 600`).

- **Limit Privileged Access:**

  - Running containers with `--privileged` grants them extended privileges.
  - Use in a secure and controlled environment.

- **Firewall and Network Policies:**

  - Implement network policies to restrict access between containers if necessary.

## License

This project is licensed under the MIT License.

