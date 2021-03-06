# LVM

![LVM/Untitled.png](LVM/Untitled.png)

## 1. Tổng quan

LVM là viết tắt của "Logical Volume Control" được sử dụng để tạo các phân vùng ảo tùy theo nhu cầu của người sử dụng. 

Ưu điểm của LVM là việc nó có tính linh hoạt và kiểm soát dễ dàng. Các tập có thể thay đổi kích thước dễ dàng khi yêu cầu không gian thay dổi.

Cấu tạo chính của LVM được mô tả như hình sau :

![LVM/Untitled%201.png](LVM/Untitled%201.png)

Trong đó có một số khái niệm cơ bản như :

- `Physical Volumes` : Ổ đĩa cứng vật lý hoặc là phân mảnh partition
- `Volume Group` : Là 1 đĩa ảo tập hợp các `Physical Volume`
- `Logical Volumes` : Là phân vùng ảo trên nền ổ đĩa ảo

Ngoài ra còn một khái niệm đó là `Extent` . Đây là một volume thuộc vào `Volume Groups` có kích thước tùy theo quy định của `Volume Group` . Nó thể hiện kích thước nhỏ nhất trên Volume. Extent trên Physical Volume là Physical Extent và tương tự đối với trên Logical volume được gọi là Logical Extent 

Từ đó ta có thể thấy Extent chính là ý tưởng cốt lõi ( core concept ) của LVM. LVM chính là cách ánh xạ giữa Physical và Logical Extent. Các Logical Extent được hợp nhất bởi LVM tạo thành Logical Volume. Nên việc giảm bớt hay thêm vào kích thước của Logical Volume chính là việc thêm hoặc bớt đi các Extent.

Sau đây chúng ta sẽ cùng thực hiện 1 số câu lệnh cơ bản để có thể xem thống số, thêm , xóa và định cỡ ( resize ) các Logical Volume thông qua LVM. 

## 2. Tạo 1 Logical Volume

Đầu tiên chúng ta sẽ quét toàn bộ thiết bị để xem ổ đĩa nào có thể tương tác với LVM thông qua lệnh `lvmdiskscan` :

```bash
[root@localhost ~]$ lvmdiskscan
  /dev/cl/root [      13.39 GiB]
  /dev/sda1    [       1.00 GiB]
  /dev/cl/swap [       1.60 GiB]
  /dev/sda2    [     <19.00 GiB] LVM physical volume
  /dev/sdb     [      16.00 GiB]
  /dev/sdc     [      16.00 GiB]
  /dev/sdd     [      16.00 GiB]
  5 disks
  1 partition
  0 LVM physical volume whole disks
  1 LVM physical volume
```

⇒ Nhận thấy rằng ta có 5 ổ đĩa , 1 phân vùng và 1 LVM Physical Volume

### Bước 1 : Tạo Physical Volume

Sau khi đã biết được tình trạng hiện tại của hệ thống , chúng ta sẽ bắt đầu tạo các Physical Volume. LVM cung cấp lệnh `pvcreate` để hỗ trợ người sử dụng tạo `Physical Volume` trên hệ thống:

```bash
[root@localhost ~]# pvcreate /dev/sdb /dev/sdc /dev/sdd
  Physical volume "/dev/sdb" successfully created.
  Physical volume "/dev/sdc" successfully created.
  Physical volume "/dev/sdd" successfully created.
```

Tiếp đó tiến hành kiểm tra lại một lần nữa các  `Physical Volume` trên hệ thống

```bash
[root@localhost ~]# pvs
  PV         VG Fmt  Attr PSize   PFree
  /dev/sda2  cl lvm2 a--  <19.00g     0
  /dev/sdb      lvm2 ---   10.00g 10.00g
  /dev/sdc      lvm2 ---   10.00g 10.00g
  /dev/sdd      lvm2 ---   10.00g 10.00g
```

Nếu bạn cần kiểm tra lại trạng thái của `Physical Volume` có thể sử dụng `pvdisplay` , ví dụ với:

```bash
[root@localhost ~]# pvdisplay /dev/sdb
  "/dev/sdb" is a new physical volume of "10.00 GiB"
  --- NEW Physical volume ---
  PV Name               /dev/sdb
  VG Name
  PV Size               10.00 GiB
  Allocatable           NO
  PE Size               0
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               sprsx7-glR2-P7w3-40FG-Fzey-ad57-VTGhA2
```

### Bước 2 : Tạo Volume Group :

![LVM/Untitled%202.png](LVM/Untitled%202.png)

Người sử dụng có thể thực hiện việc ghép các `Physical Volume` thành 1 `Volume Group` thông qua lệnh :

```bash
[root@localhost ~]# vgcreate vg0 /dev/sdb /dev/sdc
  Volume group "vg0" successfully created
```

Kiểm tra lại 1 lần nữa trạng thái của group :

```bash
[root@localhost ~]# vgdisplay vg0
  --- Volume group ---
  VG Name               vg0
  System ID
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  1
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                0
  Open LV               0
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               19.99 GiB
  PE Size               4.00 MiB
  Total PE              5118
  Alloc PE / Size       0 / 0
  Free  PE / Size       5118 / 19.99 GiB
  VG UUID               3M10Re-sNdU-3UGY-h4WO-IXV1-pOW7-NUBOBC
```

Trong đó PE có ý nghĩa là Physical Extent. Từ đó ta thấy được việc tạo các Volume thực chất là chỉ đang làm việc với các Extent mà thôi

### Bước 3 : Tạo Logical Volume

![LVM/Untitled%203.png](LVM/Untitled%203.png)

Ta thực hiện tạo 2 Volume : Project và Backup, trong đó Project có dung lượng 10Gb và Backup sẽ có dung lượng bằng 100% dung lượng còn lại

```bash
[root@localhost ~]# lvcreate -n projects -L 10G vg0
  Logical volume "projects" created.
[root@localhost ~]# lvcreate -n backups -l 100%FREE vg0
  Logical volume "backups" created.

```
```
Trong đó :  -n Tên `Logical Volume`

            -L Kích thước cụ thể của Volume

            -l Phần trăm kích thước của không gian còn lại trên Volume
```
Thực hiện kiểm tra (nếu muốn xem các thông số cụ thể của Volume thì có thể sử dụng lệnh `lvdisplay` ):

```bash
[root@localhost ~]# lvs
  LV           VG   Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root         cl   -wi-ao---- <17.00g
  swap         cl   -wi-a-----   2.00g
  backups      vg0  -wi-a-----   9.99g
  projects     vg0  -wi-a-----  10.00g
```

### Bước 4 : Tạo file hệ thống

Ở đây ta lựa chọn mount phần `Logical Volume` ra file `.ext4` do nó cho phép sự thay đổi kích thước tùy ý :

```bash
[root@localhost ~]# mkfs.ext4 /dev/vg0/projects
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
655360 inodes, 2621440 blocks
131072 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=2151677952
80 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks:
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done

[root@localhost ~]# mkfs.ext4 /dev/vg0/backups
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
655360 inodes, 2619392 blocks
130969 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=2151677952
80 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks:
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done
```

Qua 1 ví dụ đơn giản, chúng ta đã hiểu về cách thức hoạt động của LVM cũng như các cú pháp cần thực hiện khi cần tạo 1 `Logical Volume` . Có thể áp dụng tương tự nếu muốn chia thêm các `Logical Volume` khác.

## 3. Mở rộng `Volume Group` và thay đổi kích thước của `Logical Volume` :

### Bước 1 : Tạo điểm gắn kết cho quá trình định kích cỡ ( resize ) :

```bash
[root@localhost ~]# mkdir /projects
[root@localhost ~]# mkdir /backups
```

```bash
[root@localhost ~]# mount /dev/vg0/projects /projects/
[root@localhost ~]# mount /dev/vg0/backups /backups/
```

Kiểm tra dung lượng disk 

```bash
[root@localhost ~]# df -TH
Filesystem               Type      Size  Used Avail Use% Mounted on
/dev/mapper/cl-root      xfs        19G  1.1G   18G   6% /
devtmpfs                 devtmpfs  501M     0  501M   0% /dev
tmpfs                    tmpfs     512M     0  512M   0% /dev/shm
tmpfs                    tmpfs     512M  7.1M  505M   2% /run
tmpfs                    tmpfs     512M     0  512M   0% /sys/fs/cgroup
/dev/sda1                xfs       1.1G  145M  919M  14% /boot
tmpfs                    tmpfs     103M     0  103M   0% /run/user/0
/dev/mapper/vg0-projects ext4      11G   38M   9.9G   1% /projects
/dev/mapper/vg0-backups  ext4      11G   38M   9.9G   1% /backups
```

### Bước 2 : Thêm  `Physical Volume`  vào `Group Volume` :

```bash
[root@localhost ~]# vgextend vg0 /dev/sdd
  Volume group "vg0" successfully extended
```

```bash
[root@localhost ~]# vgdisplay vg0
  --- Volume group ---
  VG Name               vg0
  System ID
  Format                lvm2
  Metadata Areas        3
  Metadata Sequence No  6
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                2
  Open LV               0
  Max PV                0
  Cur PV                3
  Act PV                3
  VG Size               <29.99 GiB
  PE Size               4.00 MiB
  Total PE              7677
  Alloc PE / Size       5118 / 19.99 GiB
  Free  PE / Size       2559 / <10.00 GiB
  VG UUID               3M10Re-sNdU-3UGY-h4WO-IXV1-pOW7-NUBOBC
```

Chúng ta đã có thêm 10GB dung lượng trống, để thêm vào `Logical Volume` project ta thực hiện:

```bash
[root@localhost ~]# lvextend -l +2000 /dev/vg0/projects
  Size of logical volume vg0/projects changed from 10.00 GiB (1920 extents) to 17.81 GiB (4560 extents).
  Logical volume vg0/projects successfully resized.
```

Sau đó thực hiện tiếp thay đổi sau :

```bash
[root@localhost ~]# resize2fs /dev/vg0/projects
resize2fs 1.42.9 (28-Dec-2013)
Filesystem at /dev/vg0/projects is mounted on /projects; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 2
The filesystem on /dev/vg0/projects is now 4669440 blocks long.

[root@localhost ~]# df -TH
Filesystem               Type      Size  Used Avail Use% Mounted on
/dev/mapper/cl-root      xfs        19G  1.1G   18G   6% /
devtmpfs                 devtmpfs  501M     0  501M   0% /dev
tmpfs                    tmpfs     512M     0  512M   0% /dev/shm
tmpfs                    tmpfs     512M  7.1M  505M   2% /run
tmpfs                    tmpfs     512M     0  512M   0% /sys/fs/cgroup
/dev/sda1                xfs       1.1G  145M  919M  14% /boot
tmpfs                    tmpfs     103M     0  103M   0% /run/user/0
/dev/mapper/vg0-projects ext4       19G   47M   18G   1% /projects
/dev/mapper/vg0-backups   ext4       11G   38M  9.9G   1% /backups
```

 Vậy là chúng ta đã hoàn tất quá trình thêm dung lượng vào 1 `Logical Volume` 

## 4. Giảm kích thước của `Logical Volume` :

Việc giảm kích thước của `Logical Volume` nên thực hiện theo các bước sau để giảm thiểu lỗi :

### Bước 1 : Unmount

```bash
[root@localhost ~]# umount /projects/
[root@localhost ~]# df -TH
Filesystem              Type      Size  Used Avail Use% Mounted on
/dev/mapper/cl-root     xfs        19G  6.8G   12G  37% /
devtmpfs                devtmpfs  501M     0  501M   0% /dev
tmpfs                   tmpfs     512M     0  512M   0% /dev/shm
tmpfs                   tmpfs     512M  7.1M  505M   2% /run
tmpfs                   tmpfs     512M     0  512M   0% /sys/fs/cgroup
/dev/sda1               xfs       1.1G  145M  919M  14% /boot
tmpfs                   tmpfs     103M     0  103M   0% /run/user/0
/dev/mapper/vg0-backups ext4       11G  4.4G  5.6G  44% /backups
```

### Bước 2 : Kiểm tra lỗi :

```bash
[root@localhost ~]# e2fsck -f /dev/vg0/projects
e2fsck 1.42.9 (28-Dec-2013)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
/dev/vg0/projects: 11/1171456 files (0.0% non-contiguous), 117573/4669440 blocks
```

### Bước 3 : Thực hiện giảm kích thước file hệ thống

```bash
[root@localhost ~]# resize2fs /dev/vg0/projects 10G
resize2fs 1.42.9 (28-Dec-2013)
Resizing the filesystem on /dev/vg0/projects to 2621440 (4k) blocks.
The filesystem on /dev/vg0/projects is now 2621440 blocks long.
```

### Bước 4 : Giảm kích t hước qua `lvreduce`

```bash
[root@localhost ~]# lvreduce -L 10G /dev/vg0/projects
  WARNING: Reducing active logical volume to 10.00 GiB.
  THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce vg0/projects? [y/n]: y
  Size of logical volume vg0/projects changed from 17.81 GiB (4560 extents) to 10.00 GiB (2560 extents).
  Logical volume vg0/projects successfully resized.
```

### Bước 5: Kiểm tra file hệ thống

```bash
[root@localhost ~]# mount /dev/vg0/projects /projects/
[root@localhost ~]# lvs
  LV       VG  Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root     cl  -wi-ao---- 17.00g
  swap     cl  -wi-ao----  2.00g
  backups  vg0 -wi-ao---- 10.00g
  projects vg0 -wi-ao---- 10.00g
[root@localhost ~]# df -TH
Filesystem               Type      Size  Used Avail Use% Mounted on
/dev/mapper/cl-root      xfs        19G  6.8G   12G  37% /
devtmpfs                 devtmpfs  501M     0  501M   0% /dev
tmpfs                    tmpfs     512M     0  512M   0% /dev/shm
tmpfs                    tmpfs     512M  7.1M  505M   2% /run
tmpfs                    tmpfs     512M     0  512M   0% /sys/fs/cgroup
/dev/sda1                xfs       1.1G  145M  919M  14% /boot
tmpfs                    tmpfs     103M     0  103M   0% /run/user/0
/dev/mapper/vg0-backups  ext4       11G  4.4G  5.6G  44% /backups
/dev/mapper/vg0-projects ext4       11G  4.4G  5.6G  44% /projects
```

## 5. Xóa `Logical Volume` , `Volume Group` và `Physical Group` :

- Xóa `Logical Volume` :

```bash
$ umount /dev/new_vol_group/new_logical_volume
$ lvremove /dev/new_vol_group/new_logical_volume
```

- Xóa `Volume Group` :

```bash
$ vgremove /dev/new_vol_group
```

- Xóa `Physical Group` :

```bash
$ pvremove /dev/sdd
```

---

## Nguồn tham khảo

[LVM | Logical Volume Management | Combining Drives Together](https://www.youtube.com/watch?v=scMkYQxBtJ4)

[Giới thiệu về LVM](https://blogd.net/linux/gioi-thieu-ve-lvm/)

[Tạo và quản lý LVM trong Linux](https://blogd.net/linux/tao-va-quan-ly-lvm-trong-linux/)

[Giới thiệu về LVM - Logical Volume Management - CloudCraft](https://cloudcraft.info/gioi-thieu-ve-lvm-logical-volume-management/)

[LVM là gì? Tạo vào quản lý Logical Volume Manager (LVM) - VinaSupport](https://vinasupport.com/lvm-la-gi-tao-vao-quan-ly-logical-volume-manager/)

[Tìm hiểu Logical Volume Manager(LVM)](https://blog.cloud365.vn/linux%20tutorial/tong-quan-lvm/#itong-quan)

[Giới thiệu về Logical Volume Manager](https://bachkhoa-aptech.edu.vn/gioi-thieu-ve-logical-volume-manager.html)
