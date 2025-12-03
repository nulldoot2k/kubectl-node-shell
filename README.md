# Hướng dẫn sử dụng kubectl-node-shell

## Mục lục
1. [Giới thiệu](#giới-thiệu)
2. [Cài đặt](#cài-đặt)
3. [Cú pháp cơ bản](#cú-pháp-cơ-bản)
4. [Các tùy chọn (Options)](#các-tùy-chọn-options)
5. [Ví dụ sử dụng thực tế](#ví-dụ-sử-dụng-thực-tế)
6. [Troubleshooting](#troubleshooting)

---

## Giới thiệu

### kubectl-node-shell là gì?

`kubectl-node-shell` là một plugin cho kubectl giúp bạn truy cập vào node Kubernetes như SSH, nhưng không cần cấu hình SSH server.

### Cơ chế hoạt động

```
kubectl node-shell <node>
         ↓
Tạo privileged pod trên node
         ↓
Sử dụng nsenter để "nhảy" vào namespace của host
         ↓
Bạn có shell access vào node
         ↓
Khi thoát, pod tự động bị xóa
```

### So sánh với SSH

| Tiêu chí | SSH | kubectl-node-shell |
|----------|-----|-------------------|
| Cần SSH server | ✅ Có | ❌ Không |
| Cần mở port | ✅ Port 22 | ❌ Chỉ cần K8s API |
| Kiểm soát truy cập | SSH keys | Kubernetes RBAC |
| Tốc độ khởi tạo | ~1s | ~5-10s |
| Cleanup | Manual | Tự động |

---

## Cài đặt

### Bước 1: Download script

```bash
# Download từ GitHub (official)
curl -LO https://github.com/kvaps/kubectl-node-shell/raw/master/kubectl-node_shell

# Hoặc sử dụng script đã custom (tên pod = hacker-*)
# Copy từ artifact đã cung cấp
```

### Bước 2: Cấp quyền thực thi

```bash
chmod +x kubectl-node_shell
```

### Bước 3: Di chuyển vào PATH

```bash
# Di chuyển vào /usr/local/bin
sudo mv kubectl-node_shell /usr/local/bin/kubectl-node_shell

# Hoặc ~/bin (nếu đã có trong PATH)
mkdir -p ~/bin
mv kubectl-node_shell ~/bin/
```

### Bước 4: Kiểm tra cài đặt

```bash
# Kiểm tra version
kubectl node-shell --version
# Output: kubectl-node-shell 1.11.0

# Kiểm tra có hoạt động không
kubectl node-shell
# Output: Please specify node name
```

### Yêu cầu hệ thống

- `kubectl` đã cài đặt và cấu hình
- Quyền tạo privileged pods trong cluster
- `jq` (cho các option mount PVC)

---

## Cú pháp cơ bản

### Cú pháp tổng quát

```bash
kubectl node-shell [OPTIONS] <NODE_NAME> [-- COMMAND]
```

### Ví dụ cơ bản

```bash
# Truy cập interactive shell vào node
kubectl node-shell worker-1

# Chạy một lệnh cụ thể
kubectl node-shell worker-1 -- ls -la /var/log

# Chạy nhiều lệnh
kubectl node-shell worker-1 -- bash -c "hostname && uptime"
```

---

## Các tùy chọn (Options)

### 1. Context và Kubeconfig

```bash
# Sử dụng context cụ thể
kubectl node-shell --context production worker-1

# Hoặc format ngắn
kubectl node-shell --kubecontext=staging worker-1

# Sử dụng kubeconfig file khác
kubectl node-shell --kubeconfig ~/.kube/config-prod worker-1
```

### 2. Namespace

```bash
# Tạo pod trong namespace cụ thể
kubectl node-shell -n kube-system worker-1

# Format dài
kubectl node-shell --namespace monitoring worker-1
```

### 3. Custom Image

```bash
# Sử dụng Ubuntu thay vì Alpine
kubectl node-shell --image ubuntu:22.04 worker-1

# Sử dụng image với nhiều debug tools
kubectl node-shell --image nicolaka/netshoot worker-1

# Private registry
kubectl node-shell --image registry.company.com/debug:latest worker-1
```

### 4. Mount PersistentVolumeClaim

```bash
# Mount PVC vào /opt-pvc
kubectl node-shell -m my-pvc-name worker-1

# Trong shell, truy cập tại:
ls /opt-pvc/
```

### 5. X-Mode (Host Root Access)

```bash
# Mount toàn bộ filesystem của host vào /host
kubectl node-shell -x worker-1

# Trong shell:
ls /host/          # Root filesystem của node
ls /host/etc/
ls /host/var/lib/kubelet/
```

### 6. Namespace Isolation Options

```bash
# Không share PID namespace
kubectl node-shell --no-pid worker-1

# Không share network namespace
kubectl node-shell --no-net worker-1

# Không share IPC namespace
kubectl node-shell --no-ipc worker-1

# Không mount host filesystem
kubectl node-shell --no-mount worker-1

# Không share UTS namespace (hostname)
kubectl node-shell --no-uts worker-1

# Tắt tất cả (container bình thường)
kubectl node-shell --no-pid --no-net --no-ipc --no-mount --no-uts worker-1
```

### 7. Custom Pod Name

```bash
# Với script đã custom
kubectl node-shell worker-1
# Tạo pod: hacker-abc123

# Nếu muốn đổi prefix, edit script và sửa:
# pod="your-prefix-$(env LC_ALL=C tr -dc a-z0-9 </dev/urandom | head -c 6)"
```

### 8. Version

```bash
kubectl node-shell --version
# Output: kubectl-node-shell 1.11.0
```

---

## Ví dụ sử dụng thực tế

### Use Case 1: Troubleshooting Node NotReady

```bash
# Bước 1: Kiểm tra node status
kubectl get nodes
# NAME       STATUS     ROLES    AGE   VERSION
# worker-1   NotReady   <none>   5d    v1.28.0

# Bước 2: Vào node để debug
kubectl node-shell worker-1

# Bước 3: Kiểm tra kubelet
systemctl status kubelet
journalctl -u kubelet -n 100

# Bước 4: Kiểm tra disk space
df -h

# Bước 5: Kiểm tra container runtime
systemctl status containerd
crictl ps
```

### Use Case 2: Debug Networking Issues

```bash
# Sử dụng netshoot image (nhiều network tools)
kubectl node-shell --image nicolaka/netshoot worker-1

# Trong shell:
# Test connectivity
ping 8.8.8.8
ping kubernetes.default.svc.cluster.local

# DNS resolution
nslookup google.com
dig kubernetes.default.svc.cluster.local

# Trace route
traceroute 10.96.0.1

# Port scanning
nmap -p 6443 master-node

# Packet capture
tcpdump -i any -nn port 443

# Network performance
iperf3 -c target-server
```

### Use Case 3: Kiểm tra và Sửa Container Runtime

```bash
kubectl node-shell worker-1

# Với containerd:
crictl ps -a          # List containers
crictl images         # List images
crictl logs <id>      # View logs
crictl inspect <id>   # Inspect container

# Restart containerd nếu cần
systemctl restart containerd

# Với Docker:
docker ps
docker logs <id>
systemctl restart docker
```

### Use Case 4: Dọn dẹp Disk Space

```bash
kubectl node-shell worker-1 -x

# Tìm thư mục lớn
du -sh /host/* | sort -h | tail -10

# Xóa unused images
crictl rmi --prune

# Xóa stopped containers
crictl rm $(crictl ps -aq --state=Exited)

# Clear journal logs
journalctl --vacuum-time=7d

# Clear package cache (Ubuntu)
apt-get clean
```

### Use Case 5: Collect Debug Info

```bash
# Tạo debug bundle
kubectl node-shell worker-1 -- bash -c '
  mkdir -p /tmp/debug
  
  # System info
  hostname > /tmp/debug/hostname.txt
  uname -a > /tmp/debug/kernel.txt
  uptime > /tmp/debug/uptime.txt
  
  # Resources
  df -h > /tmp/debug/disk.txt
  free -h > /tmp/debug/memory.txt
  top -bn1 > /tmp/debug/top.txt
  
  # Kubernetes
  systemctl status kubelet > /tmp/debug/kubelet-status.txt
  journalctl -u kubelet -n 500 > /tmp/debug/kubelet.log
  crictl ps -a > /tmp/debug/containers.txt
  crictl images > /tmp/debug/images.txt
  
  # Network
  ip addr show > /tmp/debug/network.txt
  netstat -tulpn > /tmp/debug/ports.txt
  
  # Compress
  tar -czf /tmp/debug-$(hostname)-$(date +%Y%m%d-%H%M%S).tar.gz -C /tmp debug
  
  # Output location
  ls -lh /tmp/debug-*.tar.gz
'
```

### Use Case 6: Update Node Configuration

```bash
kubectl node-shell worker-1

# Sửa kubelet config
vi /var/lib/kubelet/config.yaml

# Restart kubelet để apply changes
systemctl restart kubelet

# Verify
systemctl status kubelet
journalctl -u kubelet -f
```

### Use Case 7: Certificate Issues

```bash
kubectl node-shell worker-1

# Check kubelet certificate expiry
openssl x509 -in /var/lib/kubelet/pki/kubelet.crt -text -noout | grep -A2 "Validity"

# Check all certs in /etc/kubernetes/pki
for cert in /etc/kubernetes/pki/*.crt; do
  echo "=== $cert ==="
  openssl x509 -in $cert -text -noout | grep -A2 "Validity"
done

# Renew certificates (kubeadm clusters)
kubeadm certs renew all
systemctl restart kubelet
```

### Use Case 8: Investigate Pod Failures

```bash
# Pod không start được
kubectl node-shell worker-1

# Check container logs
crictl ps -a | grep <pod-name>
crictl logs <container-id>

# Check image pull
crictl pull nginx:latest

# Check disk space for pods
df -h /var/lib/kubelet

# Check pod directory
ls -la /var/lib/kubelet/pods/
```

### Use Case 9: Performance Monitoring

```bash
kubectl node-shell worker-1 -- bash -c '
  echo "=== CPU Usage ==="
  top -bn1 | head -20
  
  echo "=== Memory Usage ==="
  free -h
  
  echo "=== Disk I/O ==="
  iostat -x 2 5
  
  echo "=== Network I/O ==="
  sar -n DEV 2 5
  
  echo "=== Load Average ==="
  uptime
'
```

### Use Case 10: Security Audit

```bash
kubectl node-shell worker-1

# Check running processes
ps aux | grep -E "(kubelet|containerd|docker)"

# Check open ports
netstat -tulpn

# Check firewall rules
iptables -L -n -v

# Check SELinux/AppArmor
getenforce  # SELinux
aa-status   # AppArmor

# Check system logs for suspicious activity
journalctl -p err -n 100
```

---

## Troubleshooting

### Lỗi: "Please specify node name"

**Nguyên nhân:** Chưa cung cấp tên node

**Giải pháp:**
```bash
# Xem danh sách nodes
kubectl get nodes

# Sử dụng đúng tên node
kubectl node-shell worker-1
```

### Lỗi: "Error from server (Forbidden)"

**Nguyên nhân:** Không có quyền tạo privileged pod

**Giải pháp:**
```bash
# Kiểm tra RBAC permissions
kubectl auth can-i create pods

# Cần admin hoặc tạo ClusterRole:
kubectl create clusterrolebinding node-shell-admin \
  --clusterrole=cluster-admin \
  --user=<your-user>
```

### Lỗi: "Unhandled Error" khi thoát

**Nguyên nhân:** Timing issue khi đóng stream

**Giải pháp:**
```bash
# Cách 1: Ignore (không ảnh hưởng)

# Cách 2: Thoát đúng cách
exit  # Chỉ gõ 1 lần, đợi vài giây

# Cách 3: Sử dụng script đã fix (trong artifact)
```

### Pod stuck trong Creating

**Nguyên nhân:** 
- Node không healthy
- Image pull failed
- Resource không đủ

**Giải pháp:**
```bash
# Kiểm tra pod status
kubectl get pods | grep hacker

# Xem chi tiết
kubectl describe pod hacker-abc123

# Xóa pod thủ công nếu cần
kubectl delete pod hacker-abc123 --force --grace-period=0
```

### Image pull error

**Nguyên nhân:** Image không tồn tại hoặc cần authentication

**Giải pháp:**
```bash
# Sử dụng image có sẵn trong node
kubectl node-shell --image alpine worker-1

# Hoặc đảm bảo imagePullSecret được cấu hình
export KUBECTL_NODE_SHELL_IMAGE_PULL_SECRET_NAME=regcred
kubectl node-shell worker-1
```

### Không thấy process của host

**Nguyên nhân:** `--no-pid` được sử dụng

**Giải pháp:**
```bash
# Bỏ --no-pid option
kubectl node-shell worker-1  # Mặc định share PID namespace
```

---

## Kết luận

`kubectl-node-shell` là công cụ mạnh mẽ để:
- ✅ Debug node issues nhanh chóng
- ✅ Không cần setup SSH
- ✅ Sử dụng Kubernetes RBAC
- ✅ Tự động cleanup
- ⚠️ Cần cẩn thận vì có full root access

**Tip cuối:** Luôn test trên môi trường non-production trước khi sử dụng trên production!

---

*Last updated: December 2024*
*Version: 1.11.0*
