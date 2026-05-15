# Containerlab Setup Guide (Ubuntu / Debian VM)
**About this Hands-On Session**
This lab has been designed for you to learn how to setup network emulation tool. The lab exercises are part of a workshop. 

This project is shared under the Creative Commons Attribution-NonCommercial 4.0 International License. For more information, visit https://creativecommons.org/licenses/by-nc/4.0/. 

<b>Prerequisites:</b> Knowledge of Ubuntu, linux commands, Virtualisation, Proxmox (optional), Orbstack (optional) and cisco concepts. 

## Official Documentation
- Containerlab Quickstart: https://containerlab.dev/quickstart/
- Containerlab Install Guide: https://containerlab.dev/install/
- Cisco IOL kind documentation: [https://containerlab.dev/manual/kinds/cisco_iol/](https://containerlab.dev/manual/kinds/cisco_iol/)
- Containerlab labs in Codespaces: https://containerlab.dev/manual/codespaces/
- vrnetlab project: https://github.com/hellt/vrnetlab

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

## Allow Docker without sudo

```bash
sudo usermod -aG docker $USER
```

Log out and back in (or reboot):

```bash
reboot
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

Download your Cisco IOL `.bin` or `.qcow2` image from Cisco (requires entitlement). Refer to https://marcstech.blog/archives/add-cisco-iol-containerlab-macos/

* Open a web browser and go to the CML Software Download page. https://software.cisco.com/download/home/286193282/type/286326381/release/CML-Free
* Click the Download icon for Cisco Modeling Labs reference platform ISO file.
* Log in with your Cisco.com (CCO ID) credentials when prompted.
* Save the Zip file (it maybe called refplat-20250616-free-iso.zip) to your Downloads folder and extract the files inside the ISO

CML Free includes:
* IOL (L3)
* IOL-L2
* ASAv (optional)

It does not include full platform images like CSR1000v or NX-OS.
For more detail about Cisco Modeling Labs (CML) https://www.cbtnuggets.com/blog/certifications/cisco/how-to-obtain-cisco-virtual-network-device-images-for-labbing

## Extract only the required IOL images

Install tools (if needed):

```bash
sudo apt update
sudo apt install -y p7zip-full
```

Extract the ISO from the ZIP:

```bash
cd ~/Downloads
7z x refplat-20250616-free-iso.zip
```

Extract only the IOL images from the ISO:

```bash
7z x refplat-20250616-free.iso \
  base-images/iol-xe-17-16-01a/x86_64_crb_linux-adventerprisek9-ms.iol \
  base-images/ioll2-xe-17-16-01a/x86_64_crb_linux_l2-adventerprisek9-ms.iol
```

You should now have:

```bash
base-images/iol-xe-17-16-01a/x86_64_crb_linux-adventerprisek9-ms.iol
base-images/ioll2-xe-17-16-01a/x86_64_crb_linux_l2-adventerprisek9-ms.iol
```

Place it in:

```
~/vrnetlab/cisco/iol/
```

## Prepare image for vrnetlab (IMPORTANT)

vrnetlab requires a specific filename format to extract the version correctly.

### IOL (Layer 3)

```bash
cp ~/Downloads/base-images/iol-xe-17-16-01a/x86_64_crb_linux-adventerprisek9-ms.iol \
   ~/vrnetlab/cisco/iol/cisco_iol-17.16.01a.bin
```

### IOL L2

```bash
cp ~/Downloads/base-images/ioll2-xe-17-16-01a/x86_64_crb_linux_l2-adventerprisek9-ms.iol \
   ~/vrnetlab/cisco/iol/cisco_iol-l2-17.16.01a.bin
```

## Build Docker image

### Build IOL (L3)
```bash
cd ~/vrnetlab/cisco/iol
make IMAGE=cisco_iol-17.16.01a.bin
```
### Build IOL L2

Clean previous file first:
```bash
rm -f cisco_iol-17.16.01a.bin
```
Then:

```bash
make docker-image
```

When complete, confirm:

```bash
docker images | grep iol
```

You should see something like:

```
vrnetlab/cisco_iol:17.16.01a   <tag>
vrnetlab/cisco_iol:L2-17.16.01a   <tag>
```
## Important notes

### 1. Filename matters 

This will NOT work:

```bash
x86_64_crb_linux-adventerprisek9-ms.iol
```

This WILL work:

```bash
cisco_iol-17.16.01a.bin
```

vrnetlab extracts the version from the filename. If it can’t, the build fails.

---

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
      image: vrnetlab/cisco_iol:17.16.01a

  nodes:
    r1:
      kind: cisco_iol
      mgmt-ipv4: 172.20.20.11
    r2:
      kind: cisco_iol
      mgmt-ipv4: 172.20.20.12

  links:
    - endpoints: ["r1:Ethernet0/1", "r2:Ethernet0/1"]
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

# Optional: Useful Commands

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

# Other resources
* https://storage.googleapis.com/site-media-prod/meetings/NANOG93/5366/20250202_Laichi_Containerlab_A_Modern_v1.pdf
* https://marcstech.blog/archives/add-cisco-iol-containerlab-macos/
* https://networkgeekstuff.com/networking/containerlab-hello-world-quick-setup-using-cisco-iol-containers/
* https://ccie-sp.gitbook.io/ccie-spv5.1-labs
* https://github.com/srl-labs/containerlab
* https://github.com/srl-labs/clab-io-draw
* Networking labs on ARM - https://youtu.be/_BTa-CiTpvI?t=1516
* Build Arista Lab on Mac or PC with ContainerLab and OrbStack - https://www.youtube.com/watch?v=RyP0Vj3igSM&list=PL-ak-hkgMCXh1DyPU1AWEtLP5KFmNtPVc
* Install GitHub CLI and Authenticate for Codespaces - https://youtu.be/NO0SW-uKveg
* ProxMox Home lab setup(playlist) - https://www.youtube.com/watch?v=gQpk_ZXnCjQ&list=PLNVfdlaRvkEmgzzaFkZh6zynXLa5IfjPY

# Cisco Virtual Router Feature Comparison

| Feature                     | **CSR1000v**      | **IOL (L3)**                | **IOL-L2**          | **IOSv**            | **XRv / XRv9000**          | **XRd**                  |
| --------------------------- | ----------------- | --------------------------- | ------------------- | ------------------- | -------------------------- | ------------------------ |
| OS Family                   | IOS-XE            | IOS                         | IOS                 | IOS                 | IOS XR                     | IOS XR                   |
| Architecture                | VM (QEMU/KVM)     | Native Linux binary         | Native Linux binary | VM                  | VM                         | Container (Docker)       |
| Boot Speed                  | Medium            | Fast                        | Fast                | Medium              | Slow                       | Fast                     |
| CPU/RAM Usage               | High              | Low                         | Low                 | Medium              | High                       | Medium                   |
| L3 Routing (OSPF/BGP/IS-IS) | ✅ Full            | ✅ Yes                       | Limited             | ✅ Yes               | ✅ Full SP stack            | ✅ Full SP stack          |
| MPLS Support                | ✅ Yes             | ⚠ Limited                   | ❌                   | ⚠ Limited           | ✅ Yes                      | ✅ Yes                    |
| L2 Switching                | ❌                 | ❌                           | ✅                   | IOSvL2 only         | ❌                          | ❌                        |
| NETCONF                     | ✅ Yes             | ❌                           | ❌                   | ⚠ Limited           | ✅ Yes                      | ✅ Yes                    |
| RESTCONF                    | ✅ Yes             | ❌                           | ❌                   | ⚠ Limited           | ❌ (XR uses different APIs) | ❌ (XR APIs)              |
| YANG Models                 | ✅ Yes             | ❌                           | ❌                   | ⚠ Partial           | ✅ Yes                      | ✅ Yes                    |
| Model-Driven Telemetry      | ✅ Yes             | ❌                           | ❌                   | ❌                   | ✅ Yes                      | ✅ Yes                    |
| Production Parity           | High (Enterprise) | Low                         | Low                 | Medium              | High (SP)                  | High (SP control-plane)  |
| Best Use Case               | Enterprise labs   | High-density CCNA/CCNP labs | L2 labs             | VIRL/CML style labs | Service provider labs      | Modern container SP labs |


