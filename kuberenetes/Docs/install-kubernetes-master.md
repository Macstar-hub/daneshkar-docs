خیلی هم عالی. برای اینکه داک را حتی از این هم ساده‌تر کنیم، بخش پیچیده‌ی Load Balancer (که برای شروع‌کننده‌ها دلهره‌آور است) را حذف می‌کنیم و به جای پروکسی APT که نیاز به فعال بودن VPN دائم دارد، از **Mirrorهای مستقیم** استفاده می‌کنیم. در این حالت، نصب پکیج‌ها بسیار سریع‌تر و راحت‌تر می‌شود.

در این نسخه، کلاستر را به صورت **Single Control Plane** (یک مغز و یک یا چند کارگر) ساده می‌کنیم تا کاربر ابتدا با محیط آشنا شود و بعداً بتواند آن را به HA ارتقا دهد.

---

# 🚀 راهنمای خیلی ساده نصب کوبرنتیز (برای شروع‌کننده‌ها)

سلام! اگر اولین بار است که می‌خواهی کوبرنتیز را نصب کنی، این راهنما دقیقاً برای توست. ما اینجا پیچیدگی‌ها را کنار گذاشتیم تا تو بتونی در سریع‌ترین زمان ممکن، اولین کلاستر خودت را بالا بیاوری.

## 🗺️ نقشه کلی ما
برای شروع، ساده‌ترین حالت را می‌سازیم: یک سرور «مغز» (Control Plane) و یک سرور «کارگر» (Worker).

| نام سرور | نقش | IP نمونه | توضیح ساده |
| :--- | :--- | :--- | :--- |
| k8s-cp-1 | Control Plane | 10.10.10.11 | مغز کلاستر (مدیریت کننده) |
| k8s-worker-1 | Worker | 10.10.10.21 | نیروی اجرایی (جایی که برنامه‌ها اجرا می‌شوند) |

**نکته:** هر جا نوشته شده «روی همه نودها»، یعنی باید روی هر دو سرور اجرا شود.

---

## 🛠️ فاز ۱: آماده‌سازی زمین (پیش‌نیازها)
*این مراحل را روی **هر دو سرور** انجام بده.*

### ۱. اسم سرورها را مشخص کنیم
روی سرور اول بزن:
```bash
sudo hostnamectl set-hostname k8s-cp-1
```
روی سرور دوم بزن:
```bash
sudo hostnamectl set-hostname k8s-worker-1
```

### ۲. سیستم‌عامل را به‌روز کنیم
```bash
sudo apt update && sudo apt upgrade -y
```

### ۳. خاموش کردن Swap (بسیار حیاتی!)
کوبرنتیز اگر Swap روشن باشد، اصلاً کار نمی‌کند. پس باید آن را کاملاً غیرفعال کنیم:
```bash
sudo swapoff -a # همین الان خاموشش کن
sudo sed -i '/swap/s/^/#/' /etc/fstab # کاری کن که بعد از ری‌استارت هم روشن نشود
# برای اطمینان بزن free -h و چک کن که جلوی Swap عدد 0 باشد.
```

### ۴. فعال کردن ماژول‌های شبکه و تنظیمات کرنل
این دستورات باعث می‌شود لینوکس اجازه دهد ترافیک شبکه بین پادها به درستی جابه‌جا شود:
```bash
# فعال کردن ماژول‌ها
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# تنظیمات شبکه
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF
sudo sysctl --system
```

---

## 📦 فاز ۲: نصب موتور اجرایی (containerd)
*این مراحل روی **همه نودها** اجرا شود.*

کوبرنتیز برای اجرای برنامه‌ها به یک موتور نیاز دارد که ما از `containerd` استفاده می‌کنیم.

### ۱. نصب containerd
```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y containerd.io
```

### ۲. تنظیمات دور زدن تحریم‌ها (خیلی مهم!)
برای اینکه ایمیج‌های کوبرنتیز بدون مشکل دانلود شوند، این تنظیمات را انجام بده:

**اول:** ساخت فایل تنظیمات:
```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

**دوم:** فعال کردن `SystemdCgroup`:
```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

**سوم:** تنظیم مسیر Mirrorها. 
در فایل `/etc/containerd/config.toml` عبارت `[plugins."io.containerd.cri.v1.images".registry]` را پیدا کن و مقدار `config_path` را به این تغییر بده:
`config_path = "/etc/containerd/certs.d"`

حالا این دستورات را بزن تا Mirrorهای جایگزین فعال شوند:

**برای ایمیج‌های کوبرنتیز:**
```bash
sudo mkdir -p /etc/containerd/certs.d/docker.io/
sudo vim /etc/containerd/certs.d/docker.io/hosts.toml
# داخل فایل بنویس:
server = "https://registry.k8s.io"
[host."https://mirror-k8s.runflare.com"]
  capabilities = ["pull", "resolve"]

sudo mkdir -p /etc/containerd/certs.d/registry.k8s.io/
sudo vim /etc/containerd/certs.d/registry.k8s.io/hosts.toml
# داخل فایل بنویس:
server = "https://registry.k8s.io"
[host."https://mirror-k8s.runflare.com"]
  capabilities = ["pull", "resolve"]
```

**برای ایمیج‌های Quay.io:**
```bash
sudo mkdir -p /etc/containerd/certs.d/quay.io
sudo vim /etc/containerd/certs.d/quay.io/hosts.toml
# داخل فایل بنویس:
server = "https://quay.io"
[host."https://quay.m.daocloud.io"]
  capabilities = ["pull", "resolve"]
```

**ری‌استارت سرویس:**
```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

اوضاع نت واقعاً روی اعصاب است! کاملاً درک می‌کنم. وقتی ریپازیتوری‌ها جواب نمی‌دهند، تنها راه مطمئن، **نصب باینری (Binary Installation)** است؛ یعنی فایل‌ها را مستقیماً دانلود کنیم و در مسیرهای سیستمی قرار دهیم.

من فاز ۳ را کاملاً بازنویسی کردم تا به جای `apt install` (که وابسته به نت و ریپازیتوری است)، فایل‌ها را مستقیماً از آدرسی که دادی دانلود و نصب کند.

---

## ☸️ فاز ۳: نصب ابزارهای اصلی کوبرنتیز (روش دستی و سریع)
*این مراحل روی **همه نودها** اجرا شود.*

چون وضعیت شبکه برای استفاده از `apt` خیلی بد است، ما ابزارهای اصلی (kubeadm, kubelet, kubectl) را به صورت مستقیم (Binary) دانلود و نصب می‌کنیم.

### ۱. دانلود مستقیم فایل‌ها
دستورات زیر را بزن تا هر سه فایل را از سرور دانلود کنی:

```bash
# دانلود kubeadm
curl -LO "https://dl.k8s.io/v1.34.9/bin/linux/amd64/kubeadm"

# دانلود kubelet
curl -LO "https://dl.k8s.io/v1.34.9/bin/linux/amd64/kubelet"

# دانلود kubectl
curl -LO "https://dl.k8s.io/v1.34.9/bin/linux/amd64/kubectl"

# دانلود kubeproxy
curl -LO "https://dl.k8s.io/release/v1.34.9/bin/linux/amd64/kube-proxy"
```

### ۲. دادن اجازه اجرا و انتقال به مسیر سیستمی
حالا باید به این فایل‌ها اجازه بدهیم که اجرا شوند و آن‌ها را به پوشه `/usr/bin` منتقل کنیم تا از هر جای سیستم بتوانیم آن‌ها را صدا بزنیم:

```bash
# دادن اجازه اجرا
chmod +x kubeadm kubelet kubectl kube-proxy

# انتقال به پوشه اجرایی سیستم
sudo mv kubeadm /usr/bin/
sudo mv kubelet /usr/bin/
sudo mv kubectl /usr/bin/
sudo mv kube-proxy /usr/bin/
```

### ۳. فعال کردن kubelet به عنوان یک سرویس
برخلاف نصب با `apt` که خودکار انجام می‌شد، در نصب دستی باید به لینوکس بگوییم که `kubelet` را به عنوان یک سرویس (Service) بشناسد و با هر بار روشن شدن سرور، خودش اجرا شود.

**اول:** ساخت فایل سرویس برای kubelet:
```bash
sudo vim /etc/systemd/system/kubelet.service
```
محتوای زیر را دقیقاً داخل این فایل کپی کن و ذخیره کن:
```ini
[Unit]
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=https://kubernetes.io/docs/
Wants=network-online.target
After=network-online.target

[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"

EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
EnvironmentFile=-/etc/default/kubelet

ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS

Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**دوم:** فعال کردن سرویس:
```bash
sudo systemctl daemon-reload
sudo systemctl enable kubelet
```

---

### 💡 یک نکته برای اطمینان:
با کامند های زیر میتوانیم صحت نصب کوبر  و بررسی کنیم
```bash
kubeadm version
kubectl version --client
kubelet --version
kube-proxy -v
```
اگر ورژن‌ها را چاپ کرد، یعنی همه چیز ردیف است و می‌توانی بری سراغ **فاز ۴ (استارت زدن کلاستر)**.

---
**راهنمای سریع برای بقیه مراحل:**
- بقیه فازها (فاز ۴ تا ۶) دقیقاً مثل قبل است.
- چون فایل‌ها را دستی ریختی، دیگر نیازی به دستور `apt-mark hold` نیست.
- فقط یادت باشد در **فاز ۴**، وقتی `kubeadm init` را می‌زنی، `kubelet` که در مرحله قبل سرویسش را فعال کردیم، شروع به کار می‌کند و کلاستر را می‌سازد.

---

## 🏁 فاز ۴: استارت زدن کلاستر (Initialize)
*این مرحله **فقط روی سرور اول (k8s-cp-1)** اجرا شود.*

حالا وقت آن است که کلاستر را بسازیم. چون Load Balancer نداریم، از IP خود سرور اول استفاده می‌کنیم.

**اجرای دستور ساخت:**
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
**خیلی مهم:** در انتهای خروجی، یک دستور می‌بینی که با `kubeadm join` شروع می‌شود. آن را کپی کن و یک جا ذخیره کن، چون برای اضافه کردن سرور Worker به آن نیاز داریم.

**تنظیم دسترسی برای مدیریت:**
```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## 🌐 فاز ۵: نصب شبکه داخلی (CNI - Calico)
تا اینجا کلاستر ساخته شده، اما نودها نمی‌توانند با هم حرف بزنند! برای این کار **Calico** را نصب می‌کنیم.

```bash
# دانلود مانیفست (مثلاً ورژن v3.32.0)
wget https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/calico.yaml

# اعمال روی کلاستر
kubectl apply -f calico.yaml
```
صبر کن تا پادها بالا بیایند: `kubectl get pods -A`. وقتی همه `Running` شدند، یعنی شبکه آماده است.

---

## 🤝 فاز ۶: اضافه کردن نیروی اجرایی (Worker Node)
*روی سرور **k8s-worker-1** برو و همان دستور `kubeadm join` که در فاز ۴ کپی کرده بودی را اجرا کن.*

مثلاً چیزی شبیه به این:
```bash
sudo kubeadm join 10.10.10.11:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

---

## ✅ تایید نهایی
حالا روی سرور اول (`k8s-cp-1`) دستور زیر را بزن:
```bash
kubectl get nodes
```
اگر هر دو سرور را در حالت **Ready** دیدی، تبریک می‌گویم! تو اولین کلاستر کوبرنتیز خودت را با موفقیت نصب کردی. 🎉