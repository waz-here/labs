# Containerlab Setup Guide (Ubuntu / Debian VM)

## Focus: Cisco IOL Lab (VM-based via vrnetlab)

This guide walks through:

1. Installing Docker and Containerlab
2. Verifying the installation
3. Building a Cisco IOL container image
4. Creating and deploying a simple lab
5. Cleaning up

Containerlab follows a **“Lab as Code”** workflow: install → write topology → deploy → inspect → destroy 

---

# Prepare Ubuntu 24.04 / Debian VM

## Update the system

```bash
sudo apt update
sudo apt -y dist-upgrade
sudo apt -y autoremove
sudo apt -y install ca-certificates curl git
```

---

# Install Docker and Containerlab

The official “all-in-one” installer installs:

* docker-ce
* containerlab
* GitHub CLI

```bash
curl -sL https://containerlab.dev/setup | sudo -E bash -s "all"
```

## Verify installation

```bash
docker run --rm hello-world
containerlab version
```

Expected:

* Docker prints “Hello from Docker”
* Containerlab version is displayed

---

## Allow Docker without sudo

```bash
sudo usermod -aG docker $USER
```

Log out and back in (or reboot):

```bash
reboot
```

Test again:

```bash
docker run --rm hello-world
```

---

# Enable Virtualisation (If Running Inside Proxmox)

Cisco IOL runs as a VM inside a container.
Nested virtualisation must be enabled.

On the Proxmox host:

```bash
qm set <VMID> --cpu host
qm start <VMID>
```

---

# Build Cisco IOL Container Image (via vrnetlab)

Containerlab integrates VM-based NOS using **vrnetlab**


## Clone vrnetlab

```bash
cd ~
git clone https://github.com/hellt/vrnetlab.git
cd vrnetlab
```


## Obtain Cisco IOL image

Download your Cisco IOL `.bin` or `.qcow2` image from Cisco (requires entitlement).

Place it in:

```
~/vrnetlab/cisco/iol/
```

Example:

```bash
cp ~/Downloads/iol-l2-adventerprisek9.bin ~/vrnetlab/cisco/iol/
```

## Build Docker image

```bash
cd ~/vrnetlab/cisco/iol
make IMAGE=iol-l2-adventerprisek9.bin
```

When complete, confirm:

```bash
docker images | grep iol
```

You should see something like:

```
vrnetlab/cisco_iol   <tag>
```


# Create a Simple Cisco IOL Lab

Create a lab directory:

```bash
mkdir -p ~/labs/iol-basic
cd ~/labs/iol-basic
```

Create a topology file:

```bash
nano iol-basic.clab.yml
```


## Example Topology (2 Cisco IOL Routers)

```yaml
name: iol-basic

mgmt:
  network: mgmt
  ipv4-subnet: 172.20.20.0/24

topology:
  kinds:
    cisco_iol:
      image: vrnetlab/cisco_iol:latest

  nodes:
    r1:
      kind: cisco_iol
      mgmt-ipv4: 172.20.20.11
    r2:
      kind: cisco_iol
      mgmt-ipv4: 172.20.20.12

  links:
    - endpoints: ["r1:Ethernet0/0", "r2:Ethernet0/0"]
```


# Deploy the Lab

```bash
sudo containerlab deploy -t iol-basic.clab.yml
```

Containerlab will:

* Create a Docker network
* Launch the IOL containers
* Connect interfaces via veth pairs 

---

# Inspect the Lab

```bash
containerlab inspect -t iol-basic.clab.yml
```

View boot logs:

```bash
docker logs -f clab-iol-basic-r1
docker logs -f clab-iol-basic-r2
```

---

# Connect to Devices

SSH into management IP:

```bash
ssh admin@172.20.20.11
ssh admin@172.20.20.12
```

Or exec directly:

```bash
docker exec -it clab-iol-basic-r1 bash
```

# Destroy the Lab

To stop and remove containers:

```bash
containerlab destroy -t iol-basic.clab.yml
```

To remove everything including lab directory:

```bash
containerlab destroy -t iol-basic.clab.yml --cleanup
```

---

# 🔎 Optional: Useful Commands

List all labs:

```bash
containerlab inspect --all
```

View the topology of the lab:

```bash
sudo containerlab graph -t iol-basic.clab.yml
```

List Docker images:

```bash
docker images
```

Remove unused images:

```bash
docker image prune
```

---

# Summary Workflow

1. Install containerlab
2. Build Cisco IOL image via vrnetlab
3. Create YAML topology
4. Deploy
5. Inspect
6. Destroy
