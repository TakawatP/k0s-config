# Kubernetes HA Cluster Installation Guide (k0s)

เอกสารฉบับนี้อธิบายขั้นตอนการติดตั้ง Kubernetes Cluster แบบ High Availability (3 Masters, 3 Workers) โดยใช้ `k0s` และ `k0sctl` พร้อมการตั้งค่า Virtual IP (VIP) สำหรับ API Server และ Ingress Controller

## 📌 Cluster Architecture
- **Load Balancer (External):** 2 VMs (HAProxy + Keepalived)
   - **VIP API Server:** 192.168.1.100 (Port 6443)
   - **VIP Web Traffic:** 192.168.1.200 (Port 80, 443)
- **Control Plane:** 3 Nodes (Master 1-3) 
- **Worker Nodes:** 3 Nodes (Worker 1-3) 
- **CNI:** Calico

---

## 🛠 Prerequisites

### 1. เครื่องที่ใช้ติดตั้ง (Local Machine)
- ติดตั้ง [k0sctl](https://github.com/k0sproject/k0sctl/releases) (Windows/Linux/macOS)
- ติดตั้ง [kubectl](https://kubernetes.io/docs/tasks/tools/)
- เตรียม SSH Private Key สำหรับรีโมทไปยัง Server ทุกเครื่อง

### 2. Server Nodes (Target Machines)
- **OS:** Ubuntu 22.04 LTS หรือใหม่กว่า (แนะนำ)
- **สิทธิ์การเข้าถึง:** `root` หรือ User ที่มีสิทธิ์ `sudo`
- **Resources:**
   - Master: 2 vCPU, 4GB RAM (ขั้นต่ำ)
   - Worker: ตามความเหมาะสมของ Workload
   - HAProxy VM: 1 vCPU, 1GB RAM
- **Network:** ปิด Firewall หรือเปิด Port ที่จำเป็น (6443, 2379-2380, 10250, 31080, 31443)

---

## 🚀 Installation Steps

### Step 1: ตั้งค่า External Load Balancer (HAProxy + Keepalived)
ทำขั้นตอนนี้บน **HAProxy VM ทั้ง 2 เครื่อง** เพื่อทำตัวเป็นทางเข้าหลักของ Cluster

1. ติดตั้ง Keepalived:
   ```bash
   sudo apt-get update && sudo apt-get install keepalived -y
2. ตั้งค่า Kernel: อนุญาตให้ Bind IP ที่ยังไม่อยู่ในเครื่อง (Non-local bind)
   ```bash
   echo "net.ipv4.ip_nonlocal_bind = 1" | sudo tee -a /etc/sysctl.conf
   sudo sysctl -p
   ```
3. แก้ไขไฟล์คอนฟิก /etc/keepalived/keepalived.conf เพื่อถือ VIP 100 และ 200
4. Configure HAProxy: แก้ไข /etc/haproxy/haproxy.cfg เพื่อ Forward Traffic ไปยัง Masters (6443) และ Workers
5. Restart Service:
   ```bash
   sudo systemctl restart haproxy
   sudo systemctl enable --now keepalived
   
### Step 2: ตรวจสอบการเชื่อมต่อ SSH

1. ตรวจสอบว่าเครื่องของคุณสามารถ SSH เข้าไปยัง Server :
   ```bash
   ssh root@192.168.1.10  # ทดสอบกับทุก IP ใน Cluster
   
### Step 3: ติดตั้ง Kubernetes Cluster ด้วย k0sctl

1. เปิด Terminal ในโฟลเดอร์ที่มีไฟล์ k0sctl.yaml แล้วรันคำสั่ง:
```bash
# สำหรับ Windows (Git Bash)
./k0sctl.exe apply --config k0sctl.yaml

# สำหรับ Linux/macOS
k0sctl apply --config k0sctl.yaml
```

### Step 4: ตั้งค่าการเข้าถึง Cluster (Kubeconfig)

1. เมื่อติดตั้งเสร็จเรียบร้อย ให้ดึงไฟล์คอนฟิกมาใช้งาน:
```bash
./k0sctl.exe kubeconfig > kubeconfig.yaml
export KUBECONFIG=$PWD/kubeconfig.yaml
```
2. ตรวจสอบสถานะ:
```bash
kubectl get nodes -o wide
```

### 🔍 Verification

ตรวจสอบสถานะของ Cluster เพื่อยืนยันความถูกต้อง:
```bash
# ตรวจสอบ Nodes
kubectl get nodes

# ตรวจสอบ Pods ของระบบ
kubectl get pods -A
```

ตรวจสอบ High Availability:
- ทดลองสั่ง stop บริการ HAProxy หรือ Keepalived บนเครื่องที่เป็น MASTER
- VIP ควรจะย้ายไปที่เครื่อง BACKUP อัตโนมัติ
- ตรวจสอบว่า kubectl get nodes ยังคงทำงานได้ปกติผ่าน VIP 192.168.1.100

### 📖 Useful Commands

ดู Logs ของ k0s บน Server:
```bash
sudo journalctl -u k0scontroller -f
```
ลบ Cluster (Reset): 
```bash
k0sctl reset --config k0sctl.yaml
```
อัปเกรด Cluster: 
แก้ไขเวอร์ชันใน k0sctl.yaml แล้วรัน apply อีกครั้ง
