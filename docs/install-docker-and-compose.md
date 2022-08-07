See:
- https://docs.docker.com/engine/install/ubuntu/
- https://docs.docker.com/compose/install/#install-compose
- https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-22-04
- https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-compose-on-ubuntu-22-04

### Install Docker
```bash
# First, update your existing list of packages:
sudo apt update

# Install a few prerequisite packages which let apt use packages over HTTPS (these packages may already be installed):
sudo apt install apt-transport-https ca-certificates curl software-properties-common

# Add the GPG key for the official Docker repository to your system:
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add the Docker repository to APT sources:
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update your existing list of packages again for the addition to be recognized:
sudo apt update

# Make sure you are about to install from the Docker repo instead of the default Ubuntu repo:
apt-cache policy docker-ce

# Output of apt-cache policy docker-ce

# docker-ce:
#   Installed: (none)
#   Candidate: 5:20.10.14~3-0~ubuntu-jammy
#   Version table:
#      5:20.10.14~3-0~ubuntu-jammy 500
#         500 https://download.docker.com/linux/ubuntu jammy/stable amd64 Packages
#      5:20.10.13~3-0~ubuntu-jammy 500
#         500 https://download.docker.com/linux/ubuntu jammy/stable amd64 Packages

# Install Docker
sudo apt install docker-ce

# Docker should now be installed, the daemon started, and the process enabled to start on boot. Check that it’s running:
sudo systemctl status docker

# If you want to avoid typing sudo whenever you run the docker command, add your username to the docker group:
sudo usermod -aG docker ${USER}

# To apply the new group membership, log out of the server and back in, or type the following:
su - ${USER}
```

### Install Docker Compose

Installing the latest version from the repository
```bash
# Update your existing list of packages:
sudo apt update

# Verify that docker-compose-plugin will be installed from the Docker repo
apt-cache policy docker-compose-plugin

# Should see something like
# docker-compose-plugin:
#   Installed: (none)
#   Candidate: 2.6.0~ubuntu-jammy
#   Version table:
#     2.6.0~ubuntu-jammy 500
#         500 https://download.docker.com/linux/ubuntu jammy/stable amd64 Packages
#     2.5.0~ubuntu-jammy 500

# Install the latest verison of the plugin (the one marked as "Candidate" in the output above (2.6.0~ubuntu-jammy))
sudo apt install docker-compose-plugin

# To install a specific version, include the version string. For example
sudo apt install docker-compose-plugin=2.5.0~ubuntu-jammy
```

Manually (the [official docs](https://docs.docker.com/compose/install/#install-compose) don't recommend this)
```bash
# Create a directory to store the docker-compose binary
mkdir -p ~/.docker/cli-plugins/

# Refer to the releases page https://github.com/docker/compose/releases for the latest version
curl -SL https://github.com/docker/compose/releases/download/v2.9.0/docker-compose-linux-x86_64 -o ~/.docker/cli-plugins/docker-compose

# Next, set the correct permissions so that the docker compose command is executable:
chmod +x ~/.docker/cli-plugins/docker-compose
```

### Ensure that Docker is behind the firewall

Docker modifies iptables directly to set up communication to and from containers. This means that UFW won’t give you a full picture of the firewall settings. You can override this behavior in Docker by adding --iptables=false to the Docker daemon

https://docs.docker.com/config/daemon/#configure-the-docker-daemon

To configure the Docker daemon using a JSON file, create a file at `/etc/docker/daemon.json`

```json
{
  "iptables": false
}
```

Restart the docker daemon
```bash
sudo systemctl restart docker.service
```