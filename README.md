# Virtualization Lab – AWS Free Tier (t3.micro / t3.small)

⚠️ **Important: Shut Down Your EC2 Instance When Finished** ⚠️
When you complete this lab, be sure to **stop or terminate your EC2 instance** from the AWS console.
Leaving it running will consume your free-tier credits and may impact your account.

---

**Goal:** Experience virtualization concepts on AWS free-tier instances without hardware-assisted virtualization (KVM). We’ll use QEMU **in emulation mode** on a `t3.micro` or `t3.small` and still practice the core skills: creating a VM disk, booting from an ISO, and installing a tiny Linux.

> **Heads-up on performance:** `t3.micro/small` do **not** expose VT-x/AMD-V to guests, so QEMU runs in **full software emulation**. Installs and boots will be slower (several minutes). That’s expected and part of the learning outcome.

---

## Prerequisites

* An AWS EC2 instance: **Ubuntu 22.04 LTS** on `t3.micro` or `t3.small` (free tier eligible)
* Security group allows SSH (port 22) to your IP
* 10–12 GB free disk space (to keep ISO + VM image)

---

## Part 0 – System Prep

Create an EC2 instance of type `t3.micro` with the Ubuntu LTS image.
You will need to create an SSH key pair to connect to your machine.

SSH into your EC2 instance and update packages:

```bash
sudo apt update && sudo apt -y upgrade
sudo reboot
```

Re-connect via SSH after the reboot.

---

## Part 1 – Verify virtualization flags (and why they’re absent)

Check if the CPU flags are visible:

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```

* On `t3.micro/small`, this should print `0`. That means **no KVM acceleration** is available here.
* We’ll proceed with QEMU **without** `-enable-kvm`.

---

## Part 2 – Install QEMU (emulation mode)

```bash
sudo apt update
sudo apt install -y qemu-system-x86 qemu-utils
```

Confirm QEMU is installed:

```bash
qemu-system-x86_64 --version
qemu-img --version
```

---

## Part 3 – Create and boot a tiny VM (Alpine Linux)

Use Alpine for smallest footprint; it boots reasonably even without KVM.

1. **Download ISO**

```bash
ALPINE_ISO=alpine-standard-3.19.1-x86_64.iso
wget https://dl-cdn.alpinelinux.org/alpine/v3.19/releases/x86_64/$ALPINE_ISO
```

2. **Create a qcow2 disk (2 GB)**

```bash
qemu-img create -f qcow2 alpine.qcow2 2G
```

3. **Boot from ISO in emulation mode**

```bash
qemu-system-x86_64 -m 768 -smp 1 \
  -drive file=alpine.qcow2,if=virtio,format=qcow2 \
  -cdrom $ALPINE_ISO -boot d -nographic
```

* At the `boot:` prompt, just press **Enter**.
* You’ll land at a shell prompt (login is usually `root` with no password in the live environment).

4. **Install to disk**

```text
setup-alpine
```

* When asked about disk, choose to **use** `vda` (virtio disk) and **sys** install.
* For all prompts, students should just accept the **defaults** (hit **Enter** each time).
* Reboot when prompted: `reboot`
* Then boot without the `-cdrom` and `-boot d` flags.

**Tip:** To exit QEMU in `-nographic` mode:
Press `Ctrl + a`, then release both keys and press `x`.
Or from another SSH session:

```bash
pkill qemu-system-x86_64
```

---

## Part 4 – Boot the installed VM (no ISO)

```bash
qemu-system-x86_64 \
  -m 768 \
  -smp 1 \
  -drive file=alpine.qcow2,if=virtio,format=qcow2 \
  -nographic
```

Log in (`root`, then the password you set during `setup-alpine`).

---

## Part 5 – Simple workload and observation

Inside the VM (Alpine):

```bash
apk update
apk add busybox-extras # for top/htop alternatives if needed
cat /etc/os-release
time sh -c "for i in 1 2 3; do echo hello $i; sleep 1; done"
```

Observe CPU usage from the EC2 host in a **second SSH session**:

```bash
top -o %CPU
```

Note the single-core emulation behavior and how it impacts performance.

---

## Deliverables (submit to Canvas)

1. Output of:

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
qemu-system-x86_64 --version
```

2. Screenshot or terminal capture showing Alpine boot or login.
3. Screenshot or terminal capture of `cat /etc/os-release && date` from the Alpine image.

---

## Notes

* This lab intentionally uses QEMU directly, without libvirt, for simplicity and transparency.
* On machines with KVM support, adding `-enable-kvm` dramatically speeds up boot and install.
* Demonstrating the speed difference between KVM-enabled and emulation-only environments reinforces the hardware-assist concept.

---

⚠️ **Final Reminder**: When you are finished, **STOP or TERMINATE your EC2 instance** from the AWS Management Console.
Leaving instances running will burn through free-tier credits and may result in charges.

```
# find all ec2 state
aws ec2 describe-instances --query "Reservations[*].Instances[*].{ID:InstanceId,State:State.Name,Name:Tags[?Key=='Name']|[0].Value}" --output table --region YOURREGION  

# Replace i-xxxxxxxxxxxxxxxxx with your actual Instance ID
aws ec2 stop-instances --instance-ids i-xxxxxxxxxxxxxxxxx --region YOURREGION
```

