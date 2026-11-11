# SRS: preload-images Feature

## 1. Bối cảnh

Khi một Kubernetes node boot từ GardenLinux image và join cluster, một trong những bước tốn thời gian nhất là pull container images (kube-proxy, CNI, pause...) từ registry. Với network chậm hoặc registry bị throttle, bước này có thể mất 1–5 phút.

Feature `preload-images` giải quyết vấn đề này bằng cách bake sẵn các images vào OS image tại build time — node boot lên là containerd đã có images, không pull gì cả.

---

## 2. Yêu cầu

| ID | Yêu cầu |
|----|---------|
| R1 | Các container images phải có sẵn trong containerd content store khi node lần đầu boot |
| R2 | Layers phải được unpack sẵn vào overlayfs snapshotter — không cần extract lúc runtime |
| R3 | Không để lại artifact build (socket, temp state) trong OS image |
| R4 | Build phải thất bại rõ ràng nếu pull image không thành công |
| R5 | Danh sách images và versions phải được pin cứng, dễ cập nhật |
| R6 | Feature phải compose được với các feature khác qua hệ thống feature của GardenLinux |

---

## 3. Thiết kế

### 3.1 Vì sao dùng `exec.post` thay vì `exec.late`

GardenLinux có 4 hook trong build pipeline:

| Hook | Chạy trong chroot? | Có network? | Có rootfs path `$1`? |
|------|-------------------|------------|----------------------|
| `exec.config` | Có | Có | Không |
| `exec.late` | Có | Có | Không |
| `exec.post` | Không (ngoài chroot) | Có | **Có** |
| image scripts | Không | Có | Có |

Để khởi động containerd daemon trỏ vào rootfs target, cần chạy từ **ngoài chroot** với path đến rootfs. Chỉ `exec.post` đáp ứng được điều kiện này.

### 3.2 Cơ chế hoạt động

```
Build container (privileged)
    │
    ├─ Build rootfs (packages, configs...)
    │
    └─ exec.post $rootfs
           │
           ├─ bind-mount /proc /sys /dev → $rootfs/proc sys dev
           │   (overlayfs snapshotter cần /proc để setup namespaces)
           │
           ├─ Khởi containerd daemon:
           │     --config  $rootfs/etc/containerd/config.toml   ← dùng config của target
           │     --root    $rootfs/var/lib/containerd            ← ghi vào rootfs
           │     --state   /tmp/containerd-preload-state-$$      ← NGOÀI rootfs
           │     --address /tmp/containerd-preload-state-$$/...  ← NGOÀI rootfs
           │
           ├─ ctr images pull --snapshotter overlayfs (mỗi image)
           │     → blobs lưu vào $rootfs/var/lib/containerd/io.containerd.content.v1.content/
           │     → snapshots unpack vào $rootfs/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/
           │
           └─ cleanup: kill containerd, umount bind mounts
                → socket và state ở /tmp, KHÔNG vào image
```

### 3.3 Tại sao phải unpack (--snapshotter overlayfs)

Nếu chỉ `import` OCI tar, containerd lưu blobs nhưng **chưa unpack**. Khi container được tạo lần đầu, containerd phải extract từng layer — mất thêm 10–30s tùy image size.

Với `ctr images pull --snapshotter overlayfs`, layers được unpack ngay tại build time thành các overlay dirs. Runtime chỉ cần mount overlay — gần như 0ms.

### 3.4 Namespace `k8s.io`

kubelet và containerd CRI đọc images từ namespace `k8s.io`. Pull phải dùng `--namespace k8s.io`, nếu không images sẽ không visible với kubelet dù đã có trong content store.

---

## 4. Cấu trúc file

```
features/preload-images/
├── info.yaml     ← khai báo feature: type element, include chost + khost
├── exec.post     ← script chính: pull + unpack images vào rootfs
└── SRS.md        ← tài liệu này
```

---

## 5. Hướng dẫn sử dụng

### 5.1 Thêm vào flavor

Mở `flavors.yaml`, thêm `preload-images` vào danh sách features của flavor muốn dùng:

```yaml
# Ví dụ: thêm cho kvm flavor có khost
- features:
    - khost
    - preload-images
    - _prod
  arch: amd64
  build: true
  test: false
  publish: true
```

> `preload-images` đã include `chost` và `khost` qua `info.yaml` nên không cần khai báo lại, nhưng khai báo thêm cũng không sao.

### 5.2 Cập nhật danh sách images

Mở `features/preload-images/exec.post`, sửa mảng `IMAGES`:

```bash
IMAGES=(
    "registry.k8s.io/pause:3.10"
    "registry.k8s.io/kube-proxy:v1.35.0"
    "quay.io/calico/node:v3.29.0"
    "quay.io/calico/cni:v3.29.0"
    "quay.io/calico/kube-controllers:v3.29.0"
)
```

Luôn pin version cụ thể (`v3.29.0`), không dùng `latest`.

### 5.3 Build image

```bash
# Từ root repo, build flavor kvm với preload-images
./build kvm-amd64

# Hoặc qua make
make kvm-amd64-build
```

### 5.4 Verify sau build

Boot VM từ image, chạy:

```bash
# Kiểm tra images đã có trong containerd
ctr --namespace k8s.io images list

# Kiểm tra snapshots đã được unpack
ctr --namespace k8s.io snapshots list
```

Expected output: tất cả images trong `IMAGES` array phải xuất hiện với status `Complete`.

### 5.5 Verify boot time

```bash
# Trên node sau khi join cluster, đo thời gian từ boot đến kubelet Ready
journalctl -u kubelet --no-pager | grep -E "Starting kubelet|Successfully registered node"

# So sánh với node không dùng preload-images
# Thông thường tiết kiệm 60-180s tùy băng thông và số images
```

---

## 6. Giới hạn và lưu ý

| Vấn đề | Chi tiết |
|--------|----------|
| **OS image size** | Tăng ~600MB–1.5GB tùy images. Calico node image một mình đã ~300MB compressed. |
| **Build time** | Tăng 2–5 phút do pull images lúc build. Nên chạy trong CI có cache registry hoặc mirror. |
| **Version drift** | Images baked vào image sẽ cũ dần. Cần rebuild OS image khi upgrade Kubernetes hoặc CNI. |
| **Registry auth** | Nếu dùng private registry, build container cần có credentials (không xử lý trong script hiện tại). |
| **Snapshotter mismatch** | Nếu runtime node dùng snapshotter khác (vd: `native`, `zfs`), snapshots sẽ không được dùng và containerd sẽ unpack lại. Mặc định GardenLinux + containerd dùng `overlayfs` nên ổn. |
| **Multi-arch** | Pull image cho arch `amd64` trên build host `arm64` cần thêm flag `--platform linux/amd64`. Cần xử lý nếu build cross-arch. |
