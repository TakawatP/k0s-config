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
