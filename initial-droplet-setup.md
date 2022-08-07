
See https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-22-04

# Initial Droplet Setup
- enable remote access to droplet with SSH
- (optional) add droplet IP and memorable name to /etc/hosts on client (i.e. your laptop)
- create non-root user
  - has root privileges 
  - has SSH access
- Setup UFW firewall
- Ensure password login is disabled
- (optional) disable `root` login


Create a new SSH key pair to enable access to new droplet with SSH.
```bash
mkdir ~/.ssh/<my-droplet-name>
ssh-keygen
```