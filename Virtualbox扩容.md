# VirtualBox 给虚拟机硬盘扩容（ centOS）

下载Gparted Live CD,一个分区管理工具。https://sourceforge.net/projects/gparted/files/gparted-live-stable/0.25.0-3/gparted-live-0.25.0-3-amd64.iso/download?use_mirror=excellmedia

 打开命令行，敲以下命令找到要扩容的虚拟机文件 。

```shell
PS D:\vbox> D:\vbox\VBoxManage.exe list hdds
UUID:           6b5d66ac-de2c-431d-bdec-baf96528dade
Parent UUID:    base
State:          created
Type:           normal (base)
Location:       D:\vbvm\atguigu\atguigu-disk001_1.vmdk
Storage format: VMDK
Capacity:       8192 MBytes
Encryption:     disabled

UUID:           abb8c419-afa2-4b51-af2e-5478b5d2ca8e
Parent UUID:    base
State:          created
Type:           normal (base)
Location:       D:\vbvm\hadoop\centos-hadoop-01\centos-hadoop-01.vdi
Storage format: VDI
Capacity:       10806 MBytes
Encryption:     disabled

UUID:           b39927c1-560d-48da-9bd5-4be87b162980
Parent UUID:    base
State:          created
Type:           normal (base)
Location:       D:\vbvm\hadoop\centos-hadoop-02\centos-hadoop-02.vdi
Storage format: VDI
Capacity:       10806 MBytes
Encryption:     disabled

UUID:           a0df8d16-446e-4ce1-8f75-b9c9d46b9ec4
Parent UUID:    base
State:          created
Type:           normal (base)
Location:       D:\vbvm\hadoop\centos-hadoop-03\centos-hadoop-03.vdi
Storage format: VDI
Capacity:       10806 MBytes
Encryption:     disabled

UUID:           6f129fb3-a909-42c4-a75f-22137e5daecd
Parent UUID:    base
State:          created
Type:           normal (base)
Location:       E:\vbvm\centos-hadoop-01-backup\centos-hadoop-01-backup.vdi
Storage format: VDI
Capacity:       10806 MBytes
Encryption:     disabled
```

这里对最后一个文件进行扩容

```shell
PS D:\vbox> D:\vbox\VBoxManage.exe modifyhd "E:\vbvm\centos-hadoop-01-backup\centos-hadoop-01-backup.vdi" --resize 20000
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
```

 这时我们发现虚拟分配空间已达到20G了，但系统并不识别。这就需要我们用到Gparted了。

分配光驱，选择已下好的虚拟光盘软件，即Gparted。 确定，启动系统，

 一直回车，语言选择26，模式选0，进入分区管理界面。选中/dev/sda3，点击工具栏，调整大小/移动。这里不能对未分配的分区进行分区，格式化，否则不能合并。 调整大小后点击Apply，退出重启。可以删除光驱启动。搞定。