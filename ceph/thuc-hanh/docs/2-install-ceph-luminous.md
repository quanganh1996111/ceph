# Cài đặt CEPH phiên bản Luminous

**Chú ý:** Phần cài đặt bản Luminous cũng tương tự như cài đặt bản Nautilus

## 1. Mô hình triển khai

![](../images/ceph-nautilus/Screenshot_1.png)

Trong đó:

- Ba node CEPH: CentOS 7 - 64bit.

- Disk: Mỗi node CEPH có 4 Disk. 1 Disk cài OS, 3 Disk lưu trữ dữ liệu cho Client.

- NICs:

    - `eth0`: SSH và cài đặt.

    - `eth1`: Kết nối thông tin giữa các node CEPH, đường cho clients kết nối.

    - `eth2`: Đồng bộ dữ liệu giữa các node CEPH.

- Phiên bản cài đặt: CEPH Nautilus.

## 2. IP Planning

![](../images/ceph-nautilus/Screenshot_2.png)

## 3. Thiết lập ban đầu

**Update OS**

```
yum install epel-release -y
yum update -y
```

**Tắt Firewall và SELinux**

```
sudo systemctl disable firewalld
sudo systemctl stop firewalld
sudo systemctl disable NetworkManager
sudo systemctl stop NetworkManager
sudo systemctl enable network
sudo systemctl start network
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```

**Cấu hình file Host**

```
cat << EOF >> /etc/hosts
10.10.40.57 ceph01
10.10.40.58 ceph02
10.10.40.59 ceph03
EOF
```

**Cài đặt NTPD**

```
yum install chrony -y
```

```
systemctl start chronyd
systemctl enable chronyd
systemctl restart chronyd
```

Kiểm tra đồng bộ thời gian:

```
chronyc sources -v
```

**Reboot lại các node CEPH**

```
init 6
```

## 4. Cài đặt CEPH

### Cài đặt ceph-deploy

```
yum install -y wget
wget https://download.ceph.com/rpm-nautilus/el7/noarch/ceph-deploy-2.0.1-0.noarch.rpm --no-check-certificate
rpm -ivh ceph-deploy-2.0.1-0.noarch.rpm
```

### Cài đặt python-setuptools để ceph-deploy có thể hoạt động ổn định.

```
curl https://bootstrap.pypa.io/ez_setup.py | python
```

### Kiểm tra cài đặt

```
ceph-deploy --version
```

Output trả về kết quả là `2.0.1` là OK

### Tạo SSH key

```
ssh-keygen
```

![](../images/ceph-nautilus/Screenshot_3.png)

**Copy ssh key sang các node khác**

```
ssh-copy-id root@ceph01
ssh-copy-id root@ceph02
ssh-copy-id root@ceph03
```

### Tạo các thư mục ceph-deploy để thao tác cài đặt vận hành cluster

```
mkdir /ceph-deploy && cd /ceph-deploy
```

### Khởi tại file cấu hình cho cụm với node quản lý là ceph01

```
ceph-deploy new ceph01
```

**Kiểm tra lại thông tin folder ceph-deploy**

![](../images/ceph-nautilus/Screenshot_4.png)

Trong thư mục bao gồm:

- `ceph.conf` : file config được tự động khởi tạo

- `ceph-deploy-ceph.log` : file log của toàn bộ thao tác đối với việc sử dụng lệnh ceph-deploy.

- `ceph.mon.keyring` : Key monitoring được ceph sinh ra tự động để khởi tạo Cluster.

- Bổ sung thêm vào file `ceph.conf`:

    - public network : Đường trao đổi thông tin giữa các node Ceph và cũng là đường client kết nối vào.

    - cluster network : Đường đồng bộ dữ liệu.

```
cat << EOF >> /ceph-deploy/ceph.conf
osd pool default size = 2
osd pool default min size = 1
osd crush chooseleaf type = 0
osd pool default pg num = 128
osd pool default pgp num = 128

public network = 10.10.40.0/24
cluster network = 10.10.41.0/24
EOF
```

### Cài đặt ceph trên toàn bộ các node ceph

**Lưu ý**: Nên sử dụng `byobu`, `tmux`, `screen` để cài đặt tránh hiện tượng mất kết nối khi đang cài đặt CEPH.

```
yum -y install byobu
```

- Cài đặt ceph lominous:

```
ceph-deploy install --release luminous ceph01 ceph02 ceph03
```

### Khởi tạo cluster với các node mon (Monitor-quản lý) dựa trên file ceph.conf

```
ceph-deploy mon create-initial
```

Sau khi thực hiện lệnh phía trên sẽ sinh thêm ra 05 file trong thư mục `ceph-deploy`

- ceph.bootstrap-mds.keyring

- ceph.bootstrap-mgr.keyring

- ceph.bootstrap-osd.keyring

- ceph.client.admin.keyring

- ceph.bootstrap-rgw.keyring

![](../images/ceph-nautilus/Screenshot_5.png)

### Để node ceph01 có thể thao tác với cluster chúng ta cần gán cho node ceph01 với quyền admin bằng cách bổ sung cho node này admin.keying

```
ceph-deploy admin ceph01
```

![](../images/ceph-nautilus/Screenshot_16.png)

## 5. Khởi tạo MGR

`Ceph-mgr` là thành phần cài đặt cần khởi tạo từ bản `lumious`, có thể cài đặt trên nhiều node hoạt động theo cơ chế Active-Passive.

### Cài đặt ceph-mgr trên ceph01

```
ceph-deploy mgr create ceph01
```

![](../images/ceph-nautilus/Screenshot_17.png)

- Truy cập vào mgr dashboard với username và password vừa đặt ở phía trên để kiểm tra.

```
https://<ip-ceph01>:7000
```

![](../images/ceph-nautilus/Screenshot_18.png)

## 6. Khởi tạo OSD

### Tạo OSD thông qua ceph-deploy tại host ceph01

Trên `ceph01`, dùng `ceph-deploy` để partition ổ cứng OSD, thay `ceph01` bằng hostname của host chứa OSD.

```
ceph-deploy disk zap ceph01 /dev/vdb
```

```
ceph-deploy osd create --data /dev/vdb ceph01
```

**Cấu hình tương tự với các Disk (Không phải Disk cài OS) còn lại trên các node.

#### Một số lưu ý

Do khởi tạo từ các VM lab trước đó trong bài Nautilus. Nên các Disk cần format lại để tránh việc cài đặt lỗi. Nếu như khởi tạo VM mới và thêm Disk mới thì có thể bỏ qua các bước này.

- Bước 1: Sử dụng lệnh `lsblk` để xác định các Disk

```
lsblk

NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda    253:0    0  20G  0 disk
├─vda1 253:1    0   1G  0 part /boot
├─vda2 253:2    0   2G  0 part [SWAP]
└─vda3 253:3    0  17G  0 part /
vdb    253:16   0  50G  0 disk
vdc    253:32   0  50G  0 disk
vdd    253:48   0  50G  0 disk
```

- Bước 2: Phân vùng lại các Disk bằng lệnh

```
parted -s /dev/vdb mklabel gpt mkpart primary xfs 0% 100%
```

- Bước 3: Reboot lại node ceph

```
reboot
```

- Bước 4: Format lại Disk

```
mkfs.xfs /dev/vdb -f
```

**Lưu ý:** thay thế `/dev/vdb` với Disk tương ứng.

## Nguồn tham khảo

https://github.com/uncelvel/tutorial-ceph/tree/master/docs/setup

https://github.com/domanhduy/ghichep/blob/master/DuyDM/CEPH/thuc-hanh/docs/2.huong-dan-cai-dat-ceph-luminous.md