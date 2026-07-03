# راه اندازی RKE2 در بستر AIR GAP (بدون اینترنت) با Cilium CNI

در این آموزش با نحوه راه‌اندازی یک کلاستر **RKE2** در محیط **Air Gap** (بدون اینترنت) همراه با **Cilium CNI** آشنا خواهید شد.

## پیشنیازها
1. ماشین‌مجازی با سیستم‌عامل `Debian 12` بدون `SWAP`
2. دسترسی `SSH` با کاربر **root** یا **sudoer** به ماشین‌های master و worker
3. فایل‌های موردنیاز برای نصب که می‌توانید از اینجا دانلود کنید: [RKE2 release page](https://github.com/rancher/rke2/releases)
4. انتقال فایل‌های دانلود شده به ماشین‌های مجازی
5. کانفیگ **MasterNode**
6. رجیستری داخلی برای دریافت ایمیج‌ها
6. نصب **rke2**
7. اضافه کردن `worker` ها
8. بررسی نهایی

### 1. ساخت ماشین‌مجازی
حداقل منابع مورد نیاز:

| Server cpu      | Server RAM | Number of agents | Storage |
| --------------- | ---------- | ---------------- | ------- | 
| 2               | 4 GB       | 0 - 255          | 20 GB   |
| 4               | 8 GB       | 226 - 450        | -----   |
| 8               | 16 GB      | 451 - 1300       | -----   |
| +16             | 32 GB      | 1300+            | -----   |

>[!TIP]
>دقت داشته باشید که ماشین‌های مجازی **نباید** *SWAP* داشته باشند

>[!TIP]
>همچنین نام ماشین‌های مجازی نباید یکسان باشند

### 2. دسترسی root یا sudoer به ماشین‌ها
روی ماشین‌های موردنظر فایل‌های `/etc/hosts/` و `/etc/hostname/` را ویرایش کنید تا نام ماشین‌ها را تغییر دهید و سپس ماشین مربوطه را ری‌استارت کنید

همچنین دسترسی `sudo` به کاربر خود در ماشین مجازی اعطا کنید یا با کاربر `root` تمام دستورات را اجرا کنید.
```bash
su - root
usermod -aG sudo <YOUR-USER>
```
### 3. دانلود فایل‌های موردنیاز
روی یک سیستم که به اینترنت دسترسی دارد از طریق صفحه [RKE2 release page](https://github.com/rancher/rke2/releases) فایل‌های موردنیاز را مطابق با نسخه موردنظر خود دانلود کنید و به ماشین‌های `MasterNode` و `WorkerNode` منتقل کنید

فایل‌های مورد نیاز:

1. `rke2.linux-amd64.tar.gz`
2. `rke2-images-core.linux-amd64.tar.zst`
3. `rke2-images-cilium.linux-amd64.tar.zst`
4. `install.sh`
5. `sha256sum-amd64.txt`

با استفاده از دستور زیر نیز میتوانید این کار را انجام دهید:

```bash
mkdir ~/rke2-artifacts && cd ~/rke2-articats
for files in "rke2.linux-amd64.tar.gz" "rke2-images-core.linux-amd64.tar.zst" "rke2-images-cilium.linux-amd64.tar.zst" "install.sh" "sha256sum-amd64.txt"; do
sudo curl -LO "https://github.com/rancher/rke2/releases/download/v1.33.1%2Brke2r1/$files"
done
```
### 4.  انتقال فایل‌های دانلود شده به ماشین‌‌های مورد نظر
به تک تک ماشین‌های مجازی که قرار است در کلاستر شما باشند ssh بزیند و دستورات زیر را اجرا کنید:

```bash
sudo mkdir -p /var/lib/rancher/rke2/agent/images/ /etc/rancher/rke2/ /root/rke2-artifacts/
```
سپس فایل‌های مورد نیاز را که دانلود کرده‌اید را منتقل کنید به دایرکتوری‌های زیر:

```bash
for ip in <MASTER-NODE> <WORKER-NODE>; do
scp ~/rke2-artifacts/* root@$ip:/root/rke2-artifacts
done
```
پس از انتقال فایل‌ها به دایرکتوری `/root/rke2-artifacts` باید فایل‌های تاربال را به `/var/lib/rancher/rke2/agent/images` نیز منتقل کنید، پس به تمام ماشین‌ها مجدد `ssh` بزنید و بعد از مطمین شدن از وجود فایل‌ها در محل `/root/rke2-artifacts`، آنها را به آدرس گفته شده کپی کنید:

```bash
sudo cp /root/rke2-artifacts/*.gz /root/rke2-artifacts/*.zst /var/lib/rancher/rke2/agent/images/
```
### 5. کانفیگ master node
با استفاده از دستور زیر یک فایل کانفیگ در مسیر زیر ایجاد کنید و به این صورت کانفیگ را وارد کنید:

>[!TIP]
> دقت داشته باشید که آدرس MasterNode خود را بگذارید

```bash
cat <<EOF | sudo tee /etc/rancher/rke2/config.yaml
write-kubeconfig-mode: "0644"
node-name: RKE2-Master-01
node-ip:
  - <MASTER-NODE-IP-ADDRESS>
cni: cilium
EOF
```
#### نکته Ip route
حالا که تمام فایل‌ها را منتقل کردید و فایل کانفیگ را ایجاد کردید نوبت به نصب می‌رسد، اما قبل از نصب حتمأ دقت داشته باشید که سیستم شما باید یک **gateway** داشته باشد، ممکن است شما نخواهید به هیچ‌عنوان ماشین‌های کلاستر شما به هیچ گیت‌وی دسترسی داشته باشند، مشکلی نیست، کافیست یک **ip** بلااستفاده و اختصاص داده نشده به هیچ ماشین یا سیستمی را بعنوان `default route` به ماشین‌های خود اختصاص دهید، با استفاده از دستور زیر:

دقت کنید که این تنظیمات را روی تمام `node`ها باید اعمال کنید

```bash
sudo ip route add default via 192.168.1.254
``` 
### 6. تنظیم رجیستری داخلی
از آنجایی که سیستم‌های شما به اینترنت دسترسی ندارند، باید از یک رجیستری داخلی برای دریافت ایمیج‌ها استفاده کنید، با فرض اینکه شما از یک رجیستری داخلی مثل `nexus` یا `jfrog` استفاده میکنید، تنظیمات را به این صورت انچام دهید:

دقت کنید که این تنظیمات را روی تمام `node`ها باید اعمال کنید
```bash
cat <<EOF | sudo tee /etc/rancher/rke2/registries.yaml
mirrors:
  docker.io:
    endpoint:
      - "https://docker.samsepiol-ss.ir"
  quay.io:
    endpoint:
      - "https://docker.samsepiol-ss.ir"
  registry.k8s.io:
    endpoint:
      - "https://docker.samsepiol-ss.ir"
  k8s.gcr.io:
    endpoint:
      - "https://docker.samsepiol-ss.ir"
  gcr.io:
    endpoint:
      - "https://docker.samsepiol-ss.ir"
  ghcr.io:
    endpoint:
      - "https://docker.samsepiol-ss.ir"
EOF
```

### 7. نصب RKE2
حالا که تمام مراحل قبل را انجام دادید، سیستم `MasterNode` شما آماده نصب می‌باشد. به ماشین `MasterNode` و سپس به `WorkerNode` متصل شوید و دستورات زیر را اجرا کنید:

دقت کنید که این تنظیمات را روی تمام `node`ها باید اعمال کنید
```bash
sudo su -
cd /root/rke2-artifacts
INSTALL_RKE2_ARTIFACT_PATH=/root/rke2-artifacts sh install.sh
```
این اسکریپت چه می‌کند:
  * فایل‌های باینری RKE2 را کپی می‌کند
  * سرویس‌های systemd را ایجاد میکند
  * ایمیج‌های دانلود شده را لود میکند

سپس باید سرویس را راه‌اندازی کنید با دستورات زیر:

برای `MasterNode`
```bash
sudo systemctl enable rke2-server.service
sudo systemctl start rke2-server.service
```
پس از نصب و راه‌اندازی کامل، در `MasterNode` فایل `kubectl` را به `/bin` منتقل کنید تا بتوانید با استفاده از این ابزار، کلاستر خود را کنترل کنید:
```bash
sudo cp ./binaries/kubectl /bin
```
سپس یک کانفیگ بسازی که `kubectl` بتواند با استفاده از آن با `API` ارتباط برقرار کند:
```bash
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
# دیدن وضعیت سلامت نودها
kubectl get nodes -o wide
kubectl -n kube-system get pods -o wide
```

#### 7.1 نصب روی WorkerNode
برای عضو کردن سایر نودهای `master` یا `worker`، باید یک توکن از `masternode-01` داشته باشیم:
```bash
cat /var/lib/rancher/rke2/server/node-token
```
**این توکن را ذخیره کنید**

حالا یک فایل کانفیگ باید برای `worker` ایجاد کنید:

>[!TIP]
> دقت کنید که توکن و آدرس Worker و Master خود را جایگزین کنید

```bash
cat <<EOF | sudo tee /etc/rancher/rke2/config.yaml
server: https://<YOUR-MASTER-01-IP>:9345
token: "YOUR-TOKEN"
node-name: RKE2-Worker-01
node-ip:
  - <YOUR-WORKER-IP>
EOF
```

حالا سرویس را با استفاده از دستورات زیر روی `worker` نیز راه‌اندازی و فعال کنید:

```bash
systemctl enable rke2-agent.service
systemctl start rke2-agent.service
```

### 8. بررسی درست بودن وضعیت
با دستورات زیر می‌توانید وضعیت کلاستر را بررسی کنید، این دستورات را روی سیستم `MasterNode` اصلی که `kubectl` را روی آن نصب کردید بزنید:

```bash
kubectl get nodes -o wide
kubectl -n kube-system get pods -o wide
```
خروچی باید به این شکل باشد:
```bash
SamSepiol@RKE2-Master-01:~$ kubectl get nodes -o wide
NAME             STATUS   ROLES                       AGE     VERSION          INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION   CONTAINER-RUNTIME
rke2-master-01   Ready    control-plane,etcd,master   5h41m   v1.33.1+rke2r1   192.168.1.160   <none>        Debian GNU/Linux 12 (bookworm)   6.1.0-29-amd64   containerd://2.0.5-k3s1
rke2-worker-01   Ready    <none>                      4h45m   v1.33.1+rke2r1   192.168.1.161   <none>        Debian GNU/Linux 12 (bookworm)   6.1.0-29-amd64   containerd://2.0.5-k3s1
```


>[!TIP]
>اگر با زدن `Tab` بعد از `kubectl` فرامین کامل نمی‌شوند، از دستور زیر استفاده کنید:

```bash
sudo apt update -y && sudo apt install bash-completion -y
echo 'source <(kubectl completion bash)' >>~/.bashrc
source ~/.bashrc
```

>[!TIP]
> اضافه کردن crictl و دیدن ایمیج‌ها

```bash
sudo rm /usr/local/bin/crictl
sudo ln -s /var/lib/rancher/rke2/bin/crictl /usr/local/bin/crictl

sudo tee /etc/crictl.yaml > /dev/null <<EOF
runtime-endpoint: unix:///run/k3s/containerd/containerd.sock
image-endpoint: unix:///run/k3s/containerd/containerd.sock
EOF
```
سپس:
```bash
sudo crictl images
```