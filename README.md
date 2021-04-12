# iSCSI Network Connection with LVM

The following information in the **"/etc/iscsi/iscsid.conf"** file is updated as specified.
```
vi /etc/iscsi/iscsid.conf
	node.startup = automatic
```
	 
Discovery of the IP address providing the iSCSI connection.
```
iscsiadm -m discovery -t st -p X.X.X.X
```

The iSCSI node information is displayed with the following command and is noted to be written in the next targetname field.
```
iscsiadm -m node
```

The connection is provided to the target providing the iSCSI connection.
```
iscsiadm -m node --targetname "iqn.1991-05.com.domain:hostname-test-target" --portal "X.X.X.X:3260" --login
```

The name of the disk whose connection and discovery has been completed is displayed.
```
lsblk
	NAME              MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
	sda                 8:0    0   150G  0 disk 
	├─sda1              8:1    0    49G  0 part /
	├─sda2              8:2    0     1K  0 part 
	└─sda5              8:5    0   975M  0 part [SWAP]
	sdb                 8:16   0   100G  0 disk 
	└─sdb1              8:17   0   100G  0 part /glusterfs
	sdc                 8:32   0 64.1TB  0 disk
```

Partitioning is done with the "parted" command.
```	
parted /dev/sdc
	GNU Parted 3.2
	Using /dev/sdc
	Welcome to GNU Parted! Type 'help' to view a list of commands.
	(parted) mklabel gpt
	(parted) unit TB
	(parted) mkpart primary 0.00TB 64.1TB
	(parted) set 1 lvm on
	(parted) print
	Model: Linux device-mapper (multipath) (dm)
	Disk /dev/sdc: 70.5TB
	Sector size (logical/physical): 512B/512B
	Partition Table: gpt
	Disk Flags:
	
	Number  Start   End     Size    File system  Name     Flags
	1      0.00TB  64.1TB  64.1TB               primary  lvm

	(parted) quit
```

The partition created with the **"fdisk -l"** command is displayed.
```
fdisk -l /dev/sdc
	Disk /dev/sdc: 64.1 TiB, 70467524231168 bytes, 137631883264 sectors
	Units: sectors of 1 * 512 = 512 bytes
	Sector size (logical/physical): 512 bytes / 512 bytes
	I/O size (minimum/optimal): 512 bytes / 512 bytes
	Disklabel type: gpt
	Disk identifier: 261A6BCC-C66C-4618-9659-5A1CF230D57C
	
	Device    Start          End      Sectors  Size Type
	/dev/sdc1  2048 125195313151 125195311104 58.3T Linux LVM
```

Physical Volume is created.
```
pvcreate /dev/sdc1
	Physical volume "/dev/sdc1" successfully created.
```	

***NOTE:** "Device /dev/sdc1 excluded by a filter." If the output is taken, the following command is run and the "pvcreate" process is applied again.*
```
wipefs -a /dev/sdc1
	/dev/sdc1: 8 bytes were erased at offset 0x00000200 (gpt): 45 46 49 20 50 41 52 54
	/dev/sdc1: 8 bytes were erased at offset 0x4016ffbffe00 (gpt): 45 46 49 20 50 41 52 54
	/dev/sdc1: 2 bytes were erased at offset 0x000001fe (PMBR): 55 aa
	/dev/sdc1: calling ioctl to re-read partition table: success
```

Virtual Group is created.
```  
vgcreate Group1 /dev/sdc1
	Volume group "Group1" successfully created
``` 
	
Logical Volume is created.
```
lvcreate -l 100%FREE -n NEWLVM Group1
	Logical volume "NEWLVM" created.
```

The group created with the **"vgdisplay Group1"** command is displayed.
```
vgdisplay Group1
	--- Volume group ---
	VG Name               Group1
	System ID
	Format                lvm2
	Metadata Areas        1
	Metadata Sequence No  2
	VG Access             read/write
	VG Status             resizable
	MAX LV                0
	Cur LV                1
	Open LV               0
	Max PV                0
	Cur PV                1
	Act PV                1
	VG Size               <58.30 TiB
	PE Size               4.00 MiB
	Total PE              15282630
	Alloc PE / Size       15282630 / <58.30 TiB
	Free  PE / Size       0 / 0
	VG UUID               7yO8ix-yitR-G2N7-bIw6-gbTw-ulde-2wVjVM
```

The lvm disk created with the **"lvdisplay"** command is displayed.
```
lvdisplay
	--- Logical volume ---
	LV Path                /dev/Group1/NEWLVM
	LV Name                NEWLVM
	VG Name                Group1
	LV UUID                amtJ1l-wXn6-XgUk-QCWb-LyU2-xOk0-YBIaMa
	LV Write Access        read/write
	LV Creation host, time HOSTNAME, 2021-03-29 15:58:24 +0300
	LV Status              available
	# open                 0
	LV Size                <58.30 TiB
	Current LE             15282630
	Segments               1
	Allocation             inherit
	Read ahead sectors     auto
	- currently set to     256
		Block device           253:18
```

The LVM disk is formatted as **"ext4"**.
```
mkfs.ext4 /dev/mapper/Group1-NEWLVM
```

UUID information of the disk is learned with the **'blkid | grep "Group1-NEWLVM"'** command.
```
blkid | grep "Group1-NEWLVM"
	/dev/mapper/Group1-NEWLVM: UUID="25c13348-6751-4a17-817d-948424175089" TYPE="ext4"
```

The learned UUID information is added into **fstab**.
```
vi /etc/fstab
	UUID=25c13348-6751-4a17-817d-948424175089	/data	ext4    errors=continue 0       0
```

The **"/data"** directory is created and mounted.
```
mkdir /data && mount /data && df -h
```
<br><br>
### Mounting to Other Servers

The connection is provided to the target providing the iSCSI connection.
```
iscsiadm -m node --targetname "iqn.1991-05.com.domain:hostname-test-target" --portal "X.X.X.X:3260" --login
```

A rescan is performed to detect the disk added on other servers.
```
iscsiadm -m session --rescan
```
	
After the server has restarted, the mount process is performed by applying the **'blkid'** and the following steps in the previous heading.
<br><br><br><br>
### Extending the LVM Disk

It is ensured that the newly defined disk is detected.

The name of the disk whose connection and discovery has been completed is displayed.
```
lsblk
	NAME              MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
	sda                 8:0    0   150G  0 disk 
	├─sda1              8:1    0    49G  0 part /
	├─sda2              8:2    0     1K  0 part 
	└─sda5              8:5    0   975M  0 part [SWAP]
	sdb                 8:16   0   100G  0 disk 
	└─sdb1              8:17   0   100G  0 part /glusterfs
	sdd                 8:32   0 64.1TB  0 disk
```
	
Partitioning is done with the **"parted"** command.
```	
parted /dev/sdd
	GNU Parted 3.2
	Using /dev/sdd
	Welcome to GNU Parted! Type 'help' to view a list of commands.
	(parted) mklabel gpt
	(parted) unit TB
	(parted) mkpart primary 0.00TB 64.1TB
	(parted) set 1 lvm on
	(parted) print
	Model: Linux device-mapper (multipath) (dm)
	Disk /dev/sdd: 70.5TB
	Sector size (logical/physical): 512B/512B
	Partition Table: gpt
	Disk Flags:
	
	Number  Start   End     Size    File system  Name     Flags
	1      0.00TB  64.1TB  64.1TB               primary  lvm
	
	(parted) quit
```

Physical Volume is created.
```
pvcreate /dev/sdd1
	Physical volume "/dev/sdd1" successfully created.
```

The following command is run to extend the existing group.
```
vgextend Group1 /dev/sdd1
```	
	
The following command is run to increase the existing logical volume space.
```
lvextend -l +100%FREE /dev/mapper/Group1-NEWLVM
```

The following command is run to resize the logical volume area.
```
resize2fs /dev/mapper/Group1-NEWLVM
```

With the **"df"** command, it is seen that the disk size is increased.
```
df -h
```
