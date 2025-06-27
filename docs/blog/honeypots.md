## Why I Like Internet Storm Center & DShield (and How You Can Help Too)

I'm a big fan of the **SANS Internet Storm Center (ISC)** and its community-driven mission to track global internet threats. They publish daily handler diaries, offer practical incident response tools, and provide a platform where people like you and me can contribute valuable data. One of the most impactful ways to do that is by running a **DShield Honeypot**.

DShield is a **low-interaction honeypot** that collects SSH, Telnet, and HTTP scan data. It’s lightweight, informative, and most importantly, it shares that telemetry with ISC to help map emerging threats. Setting one up on AWS is a great weekend project, especially if you're into networking, cloud, or cybersecurity.

Below is a step-by-step guide I followed and adapted from [Matthew OB's post](https://matthewob5.medium.com/setting-up-a-dshield-honeypot-in-aws-2ca5f8a29d9).

---

## Comprehensive Guide: Setting Up a DShield Honeypot on AWS (Ubuntu 22.04 LTS)

### What is DShield?

**DShield** is a honeypot system built by ISC to collect logs from distributed sensors. It:

- Captures SSH/Telnet login attempts (via Cowrie)
- Records HTTP requests and firewall activity
- Sends anonymized logs to ISC

### Requirements

- AWS Free Tier account  
- DShield account: [https://secure.dshield.org/myaccount.html](https://secure.dshield.org/myaccount.html)  
- SSH key pair  
- Basic Linux/terminal experience

**Tips:**

- Enable MFA on AWS
- Use IAM users instead of root
- Set Free Tier alerts to avoid surprises

### Step 1: Launch EC2 Instance

1. Go to EC2 → Launch Instance  
2. Name: `DShieldHoneypot`  
3. AMI: **Ubuntu Server 22.04 LTS** (not 24.04!)  
4. Type: `t2.micro`  
5. Key pair: Use or create `.pem` file  
6. Storage: 25 GB  
7. Security Group: Allow SSH (port 22) from `YOUR_PUBLIC_IP/32`

### Step 2: Connect & Update

```bash
chmod 400 your-key.pem
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>

sudo apt update && sudo apt upgrade -y
sudo reboot
```

Reconnect after reboot.

### Step 3: Create Honeypot User & Harden Sudo

```bash
sudo passwd ubuntu
sudo adduser --disabled-password --gecos "DShield Honeypot" dshield
sudo visudo /etc/sudoers.d/90-cloud-init-users
```

Change this line:
```text
ubuntu ALL=(ALL) NOPASSWD:ALL
```
To:
```text
ubuntu ALL=(ALL) ALL
```

### Step 4: Install Dependencies

```bash
sudo apt install python3-pip python2.7 git -y
curl https://bootstrap.pypa.io/pip/2.7/get-pip.py --output get-pip.py
sudo python2.7 get-pip.py
sudo reboot
```

### Step 5: Install DShield

```bash
mkdir ~/install && cd ~/install
git clone https://github.com/DShield-ISC/dshield.git
cd dshield/bin
sudo ./install.sh
```

#### During setup:

- Accept privacy warnings
- Enable automatic updates
- Enter your DShield email + API key
- Interface: `eth0`
- Admin Port: 12222 (default)
- Local Network: `172.16.0.0/12`
- Ignore FW logs from your public IP
- Accept default SSL values
- Exclude ports: 2222, 2223, 8000

### Step 6: Confirm SSH on Port 12222

```bash
sudo ss -tuln | grep 12222
sudo systemctl restart ssh  # if not open
sudo ss -tuln | grep 12222
```

Update your security group to allow TCP 12222 from your IP, then test:

```bash
ssh -p 12222 -i your-key.pem ubuntu@<EC2_PUBLIC_IP>
```

Once confirmed:
```bash
sudo reboot
```

### Step 7: Configure Security Group

#### Inbound Rules:

| Port Range      | Protocol | Source             | Purpose             |
|------------------|----------|----------------------|----------------------|
| 12222            | TCP      | YOUR_PUBLIC_IP/32    | SSH Management       |
| 0-12221          | TCP/UDP  | 0.0.0.0/0            | Honeypot Listeners   |
| 12223-65535      | TCP/UDP  | 0.0.0.0/0            | Honeypot Listeners   |
| All ICMP         | ICMP     | 0.0.0.0/0            | Ping/Traceroute      |

#### Outbound Rules:

| Port   | Protocol | Destination | Purpose         |
|--------|----------|-------------|-----------------|
| All    | All      | 0.0.0.0/0   | Allow all traffic |

Make sure the Security Group is associated with the instance, and apply NACLs if needed.

### Step 8: Verify Operation

```bash
cd ~/install/dshield/bin
./status.sh
```

You should see:
```text
Submitted to DShield: N log(s) sent.
All services running.
```

### Step 9: Monthly Maintenance

```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
cd ~/install/dshield/bin
./update.sh
```

### Step 10: View Logs on DShield

Visit: [https://secure.dshield.org/myaccount.html](https://secure.dshield.org/myaccount.html)

You’ll find:
- SSH Logs
- Web Logs
- Firewall Events

---

## You're Done!

Your honeypot is now online and feeding data into ISC’s global threat map. You've helped contribute to public good in the infosec community and learned a lot in the process. 👏

Check out [Matthew OB’s original guide](https://matthewob5.medium.com/setting-up-a-dshield-honeypot-in-aws-2ca5f8a29d9) for visuals or troubleshooting.

