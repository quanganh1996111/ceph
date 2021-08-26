# Cài đặt và tích hợp CEPH với OpenStack

## 1. IP Planning

![](../images/ceph-openstack/Screenshot_1.png)

- Tham khảo việc cài đặt 3 Node CEPH [tại đây](https://github.com/quanganh1996111/ceph/tree/main/ceph/thuc-hanh/docs)

- Tham khảo việc cài đặt OpenStack gồm 1 node Controller và 2 node Compute [tại đây](https://github.com/quanganh1996111/openstack/blob/main/install-openstack/docs/1-install-openstack-manual.md)

## 2. Các bước triển khai

Về cơ bản Openstack có 3 service có thể tích hợp để lưu trữ xuống dưới CEPH.

- Glance (images): Lưu trữ các images xuống CEPH.

- Cinder (volume): Lưu trữ các ổ đĩa của máy ảo được tạo ra trên Openstack xuống dưới CEPH.

- Nova (conmpute): Mặc định khi VM được tạo sẽ sinh ra một file lưu thông tin file disk của VM đó trên chính node compute đó, có thể tích hợp xuống CEPH thông qua một `symlink`.

## 3. Phần Chuẩn bị cài đặt.

Thực hiện chung cho việc tích hợp các thành phần Openstack vào CEPH.

### 3.1. Cài đặt lib ceph python cho các node Controller và Compute

- `python-rbd`: Thư viện hỗ trợ ceph.

- `ceph-common`: Kết nối client vào ceph cluster.

```
yum install -y python-rbd ceph-common
```

### 3.2. Tạo pool trên CEPH (Thực hiện trên node CEPH)

Tùy thuộc và nhu cầu tích hợp để tạo những pool khác nhau với mục đích quản lý khác nhau. Sử dụng công cụ tính toán pool do CEPH hỗ trợ.

https://linuxkidd.com/ceph/pgcalc.html

- Tạo pool theo tài nguyên của cụm CEPH

![](../images/ceph-openstack/Screenshot_2.png)

- Generate ra lệnh để tiến hành cài đặt:

![](../images/ceph-openstack/Screenshot_3.png)

- Chạy các câu lệnh ở trong file vừa được Generate để khởi tạo pool trên CEPH

```
ceph osd pool create volumes 1024
ceph osd pool set volumes size 2
while [ $(ceph -s | grep creating -c) -gt 0 ]; do echo -n .;sleep 1; done

ceph osd pool create backup 64
ceph osd pool set backup size 2
while [ $(ceph -s | grep creating -c) -gt 0 ]; do echo -n .;sleep 1; done

ceph osd pool create vms 64
ceph osd pool set vms size 2
while [ $(ceph -s | grep creating -c) -gt 0 ]; do echo -n .;sleep 1; done

ceph osd pool create images 128
ceph osd pool set images size 2
while [ $(ceph -s | grep creating -c) -gt 0 ]; do echo -n .;sleep 1; done
```

- Thông báo lỗi khi tạo pool:

![](../images/ceph-openstack/Screenshot_4.png)

Mặc định CEPH không cho xóa pool để tránh rủi ra mất mát data khi xóa nhầm pool. Để có thể xóa được pool phải chỉnh sửa file ceph config trong thư mục ceph-deploy và overwrite config tới các node khác thông qua ceph-deploy.

Chỉnh sửa cấu hình trong file `/ceph-deploy/ceph.conf`

Thêm cấu hình:

```
mon_allow pool delete = true
```

- Muốn giới hạn số lượng PG trên OSDs, ta thêm:

```
mon max pg per osd  = 500

hoặc

mon_max_pg_per_osd = 500
```

Trong config của CEPH có thể có dấu _ hoặc không có đều OK.

Restart service ceph MON, ceph MGR.

```
systemctl restart ceph-mon@ceph01
systemctl restart ceph-mgr@ceph01
```

**Sau mỗi lần chỉnh sửa config hoặc key chỉ cần đứng trong thực mục ceph-deploy copy đè config lên các node CEPH khác**

- Copy đè key sang các node CEPH khác:

```
ceph-deploy --overwrite-conf admin ceph01 ceph02 ceph03
```

- Copy đè `config`

```
ceph-deploy --overwrite-conf config push ceph01 ceph02 ceph03
```

- Tiến hành khởi tạo lại các Pool như trên. Kết quả:

![](../images/ceph-openstack/Screenshot_5.png)

![](../images/ceph-openstack/Screenshot_6.png)