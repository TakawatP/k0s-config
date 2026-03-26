# Kubernetes HA Cluster Installation Guide (k0s)

เอกสารฉบับนี้อธิบายขั้นตอนการติดตั้ง Kubernetes Cluster แบบ High Availability (3 Masters, 3 Workers) โดยใช้ `k0s` และ `k0sctl` พร้อมการตั้งค่า Virtual IP (VIP) สำหรับ API Server และ Ingress Controller

## 📌 Cluster Architecture
- **Control Plane:** 3 Nodes (Master 1-3) + Virtual IP (`192.168.1.100`)
- **Worker Nodes:** 3 Nodes (Worker 1-3) + Virtual IP (`192.168.1.200`)
- **CNI:** Calico
- **Ingress:** Ingress-NGINX (Port 31080 / 31443)

---

## 🛠 Prerequisites

### 1. เครื่องที่ใช้ติดตั้ง (Local Machine)
- ติดตั้ง [k0sctl](https://github.com/k0sproject/k0sctl/releases) (Windows/Linux/macOS)
- ติดตั้ง [kubectl](https://kubernetes.io/docs/tasks/tools/)
- เตรียม SSH Private Key สำหรับรีโมทไปยัง Server ทุกเครื่อง

### 2. Server Nodes (Target Machines)
- OS: Ubuntu 22.04 LTS หรือใหม่กว่า (แนะนำ)
- สิทธิ์การเข้าถึง: `root` หรือ User ที่มีสิทธิ์ `sudo`
- ปิด Firewall หรือเปิด Port ที่จำเป็น (6443, 2379-2380, 10250, 31080, 31443)

---

## 🚀 Installation Steps

### Step 1: ตั้งค่า High Availability (Keepalived)
รันคำสั่งเหล่านี้บน **Server ทุกเครื่อง** เพื่อสร้าง Virtual IP

1. ติดตั้ง Keepalived:
   ```bash
   sudo apt-get update && sudo apt-get install keepalived -y
2. แก้ไขไฟล์คอนฟิก /etc/keepalived/keepalived.conf (ตามตัวอย่างในโปรเจกต์)
3. Restart Service:
   ```bash
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

### 🔍 Verification

ตรวจสอบสถานะของ Cluster เพื่อยืนยันความถูกต้อง:
```bash
# ตรวจสอบ Nodes
kubectl get nodes

# ตรวจสอบ Pods ของระบบและ Ingress
kubectl get pods -A

# ตรวจสอบ Ingress Service (Port 31080/31443)
kubectl get svc -n ingress-nginx
```

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
