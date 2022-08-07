See:
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

```bash
# Create a directory to store the docker-compose binary
mkdir -p ~/.docker/cli-plugins/

# Refer to the releases page https://github.com/docker/compose/releases for the latest version
curl -SL https://github.com/docker/compose/releases/download/v2.9.0/docker-compose-linux-x86_64 -o ~/.docker/cli-plugins/docker-compose

# Next, set the correct permissions so that the docker compose command is executable:
chmod +x ~/.docker/cli-plugins/docker-compose
```

### Disable something in docker daemon to make UFW work
https://docs.docker.com/config/daemon/#configure-the-docker-daemon

Docker modifies iptables directly to set up communication to and from containers. This means that UFW won’t give you a full picture of the firewall settings. You can override this behavior in Docker by adding --iptables=false to the Docker daemon

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