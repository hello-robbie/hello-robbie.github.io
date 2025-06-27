## Why I Like Internet Storm Center & DShield (and How You Can Help Too)

I'm a big fan of the **SANS Internet Storm Center (ISC)** and its community-driven mission to track global internet threats. They publish daily handler diaries, offer practical incident response tools, and provide a platform where people like you and me can contribute valuable data. One of the most impactful ways to do that is by running a **DShield Honeypot**.

DShield is a **low-interaction honeypot** that collects SSH, Telnet, and HTTP scan data. It’s lightweight, informative, and most importantly, it shares that telemetry with ISC to help map emerging threats. Setting one up on AWS is a great weekend project, especially if you're into networking, cloud, or cybersecurity.

Below is a step-by-step guide I followed and adapted from [Matthew OB's post](https://matthewob5.medium.com/setting-up-a-dshield-honeypot-in-aws-2ca5f8a29d9).

---

# Comprehensive Guide: Setting Up a DShield Honeypot on AWS (Ubuntu 22.04 LTS)
---

## 1. Introduction

This guide walks through setting up a **DShield low-interaction honeypot** using Ubuntu Server 22.04 LTS on an AWS EC2 instance. It includes practical steps and insights from:  
https://matthewob5.medium.com/setting-up-a-dshield-honeypot-in-aws-2ca5f8a29d9

---

## 2. What is DShield?

**DShield** is a low-interaction honeypot created by the SANS Internet Storm Center (ISC). It:
- Collects SSH and Telnet login attempts (via Cowrie)
- Captures HTTP requests
- Logs firewall activity
- Sends logs to ISC for global correlation

---

## 3. Requirements

- AWS Free Tier account  
- DShield account: https://secure.dshield.org/myaccount.html  
- SSH key pair  
- Familiarity with Linux terminal

**Tips:**
- Enable MFA on AWS accounts
- Use IAM users, not root
- Set Free Tier alerts

---

## 4. Launch EC2 Instance

Use **Ubuntu Server 22.04 LTS**. Do *not* use 24.04, there are differences in 24.04 that break DShield.

1. Go to EC2 > Launch Instance  
2. Name: `DShieldHoneypot`  
3. AMI: Ubuntu Server 22.04 LTS  
4. Instance type: `t2.micro`  
5. Key pair: Use or create `.pem` file  
6. Storage: 25 GB  
7. Security Group:
   - Inbound: Allow SSH (port 22) from **Your_Public_IP/32**

---

## 5. SSH into EC2 & Update
Default SSH Credentials:
* Username: `ubuntu`
* Password: *None* (login is done via SSH key only - password authentication is disabled by default)

Use the command below to connect:
```
chmod 400 your-key.pem
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>
```
Use the command below to connect:
```
sudo apt update && sudo apt upgrade -y
sudo reboot
```
Reconnect after reboot.

---

## 6. Create Honeypot User
Before tightening `sudo` permissions, **set a password for the ``user to avoid locking yourself out:
```
sudo passwd ubuntu
```

You'll be prompted to enter and confrm a new password. Save it in a secure location. Then, create a dedicated, non-login user:
```
sudo adduser --disabled-password --gecos "DShield Honeypot" dshield
```

Lock down sudo access by editing sudoers:

`sudo visudo /etc/sudoers.d/90-cloud-init-users`

Change:
`ubuntu ALL=(ALL) NOPASSWD:ALL`

To:
`ubuntu ALL=(ALL) ALL`

---

## 7. Install Dependencies
Run each command separately to spot issues:

`sudo apt install python3-pip python2.7 git -y`
`curl https://bootstrap.pypa.io/pip/2.7/get-pip.py --output get-pip.py`
`sudo python2.7 get-pip.py`

Note: `python-pip` is no longer available - use the pip bootstrap script for Python 2.

Reboot the server again:
`sudo reboot`

---

## 8. Install DShield
Setup the installation environment and run the install script

```
mkdir ~/install && cd ~/install
git clone https://github.com/DShield-ISC/dshield.git
cd dshield/bin
sudo ./install.sh
```

The installer will walk you through configuration:

1. Accept risk and privacy notices

2. Choose Automatic Updates

3. Enter your DShield email and API key > Click Verify

4. Select network interface (usually `eth0`)

5. Configure:
   - Admin Port: Keep default `12222` or choose another
   - Local Network: Use `172.16.0.0/12` if this is your only instance or the full VPC CIDR
   - Additional IPs: Add your public IP

6. Add the same IPs under Ignore FW Log (they won't be logged or redirected)

7. SSL Certificate:

   - Accept default fields: US / Florida / Jacksonville / DShield / Decoy

   - Let it generate a new CA

8. Configure Honeypot Exceptions:
   - Add same IPs to disable honeypot for (internal testing IPs)
   - Add ports `2222`, `2223`, and `8000` to exclude from redirection and logging
  
**Watch Out:** If you enter incorrect IPs or CIDRs during this step (e.g., mistyping your home IP or setting a wrong subnet), you may block yourself from SSH access on the admin port. If this happens, update the AWS Security Group to temporarily allow access again.

**Watch Out:** The DShield installer modifies /etc/ssh/sshd_config to change the SSH port to 12222. However, this change does not always take effect immediately. Before rebooting:

1. Run `sudo ss -tuln | grep 12222` — confirm the port is open.

2. If it’s not open, run:

`sudo systemctl restart ssh`
`sudo ss -tuln | grep 12222`

3. Confirm success before continuing.

4. Update the security group attached to the EC2 instance, not just the subnet-level ACLs. The SG must allow TCP 12222 inbound from your IP.

Then, test SSH access from your local machine in a separate terminal:

```ssh -p 12222 -i /path/to/your-key.pem ubuntu@<EC2_PUBLIC_IP>```

Only proceed with the reboot after confirming that SSH on port 12222 is working.

Then reboot:
`sudo reboot`

--

## 9. Update Security Group
Update your Security Group to expose the honeypot to the internet while restricting management access.

**Recommended Inbound Rules:**

| Type              | Protocol | Port Range    | Source             | Description                 |
|-------------------|----------|---------------|---------------------|-----------------------------|
| Custom TCP        | TCP      | 12222         | <Your_Public_IP>/32  | Allow SSH from home         |
| Custom TCP        | TCP      | 0-12221       | 0.0.0.0/0          | Expose honeypot ports       |
| Custom TCP        | TCP      | 12223-65535   | 0.0.0.0/0          | Expose honeypot ports       |
| Custom UDP        | UDP      | 0-12221       | 0.0.0.0/0          | Expose honeypot ports       |
| Custom UDP        | UDP      | 12223-65535   | 0.0.0.0/0          | Expose honeypot ports       |
| All ICMP - IPv4   | ICMP     | All           | 0.0.0.0/0          | Allow ping/traceroute       |
| All ICMP - IPv6   | ICMPv6   | All           | ::/0               | Allow IPv6 ping             |
| (Same TCP/UDP)    | TCP/UDP  | All ranges    | ::/0               | Matching IPv6 exposure      |


**Outbound Rules:**
| Type         | Port | Destination | Description        |
|--------------|------|-------------|--------------------|
| All traffic  | All  | 0.0.0.0/0   | Allow outbound     |

**Update Inbound ACLs**
| Rule # | Type        | Protocol | Port Range | Source             | Action |
|--------|-------------|----------|------------|---------------------|--------|
| 100    | All traffic | All      | All        | 0.0.0.0/0           | Allow  |
| 110    | Custom TCP  | TCP (6)  | 12222      | <Your_Public_IP>/32   | Allow  |
| *      | All traffic | All      | All        | 0.0.0.0/0           | Deny   |

**Update Outbound ACLs**
| Rule # | Type        | Protocol | Port Range   | Destination         | Action |
|--------|-------------|----------|--------------|----------------------|--------|
| 100    | All traffic | All      | All          | 0.0.0.0/0            | Allow  |
| 110    | Custom TCP  | TCP (6)  | 1024-65535   | <Your_Public_IP>/32    | Allow  |
| *      | All traffic | All      | All          | 0.0.0.0/0            | Deny   |


**Watch Out:** Creating the correct security group is not enough - you must **associate it** with the instance:
1. Go to **EC2 -> Instances**
2. Select your honeypot instance
3. Scroll to the **Security** tab
4. Click **Manage Security Groups**
5. Ensure your SG is added
Note: AWS allows traffic if **any** associated SG allows it.

**Watch Out:** Creating the ACL is not enough — you must associate it with the subnet:
1. Go to VPC → Network ACLs
2. Select your honeypot ACL
3. Click the “Subnet associations” tab
4. Click Edit subnet associations
5. Check the box for the subnet your honeypot EC2 instance is using
6. Save your changes
Note: NACLs apply at the subnet level and evaluate both inbound and outbound rules. The most specific (lowest rule number) is evaluated first. Be sure your allow rules come before the default deny.

**What if your IP changes?** If your public IP changes and you've restricted `port 12222` to your old IP, you'll lose SSH access. To fix this:
1. Log into the AWS Console.
2. Go to EC2 > Security Groups.
3. Find your honeypot's SG and edit the inbound rules.
4. Update the Custom TCP rule for port `12222` to reflect `<your new public IP>` (use whatismyip.com to find it).

---

## 10. Verify Status
Run the status script:
`cd ~/install/dshield/bin`
`./status.sh`

Look for messages like:
```
Submitted to DShield: N log(s) sent.
All services running.
```

Reboot the server if there are any errors. Otherwise refer to https://github.com/DShield-ISC/dshield/blob/main/STATUSERRORS.md

---

## 11. Maintenance
It’s a good idea to log in once a month to run `sudo apt update && sudo apt upgrade -y`, followed by a **system reboot**. Running the `update.sh` script monthly is also recommended to ensure your honeypot stays current.
`sudo apt update && sudo apt upgrade -y`
`sudo reboot`
`cd ~/install/dshield/bin`
`./update.sh`

---

## 12. View Logs
After some time, log into your DShield account ahd check:
- SSH Logs
- Web Logs
- Firewall Events

URL: https://secure.dshield.org/myaccount.html

## Done
Your AWS-hosted DShield honeypot is now online and feeding threat data to the Internet Storm Center. You've also locked it down properly and added safety measures to avoid accidental lockouts.

For screenshots and visuals, refer to [Matthew’s blog post.](https://matthewob5.medium.com/setting-up-a-dshield-honeypot-in-aws-2ca5f8a29d9)
