# Infrastructure setup


## Set up DHCP server

```
sudo apt-get install isc-dhcp-server
```

```
--> /etc/dhcp/dhcpd.conf

# common options
option domain-name "brucejacobs.org";
option domain-name-servers 192.168.86.15;
default-lease-time 300;
max-lease-time 7200;

authoritative;
infinite-is-reserved on;

# lab subnet
subnet 192.168.86.0 netmask 255.255.255.0 {
  range 192.168.86.50 192.168.86.80;
  option routers 192.168.86.1;
}

# static reservations

host haproxy1 {
  hardware ethernet 00:0c:29:eb:88:bc;
  fixed-address 192.168.86.18;
}

host media {
  hardware ethernet 00:0c:29:3f:dc:fe;
  fixed-address 192.168.86.16;
}

host k8s-controller1 {
  hardware ethernet 00:0c:29:ef:d0:2f;
  fixed-address 192.168.86.11;
}

host k8s-controller2 {
  hardware ethernet 00:0c:29:42:15:8c;
  fixed-address 192.168.86.12;
}

host k8s-controller3 {
  hardware ethernet 00:0c:29:df:e9:81;
  fixed-address 192.168.86.13;
}

host k8s-worker1 {
  hardware ethernet 00:0c:29:9a:20:6d;
  fixed-address 192.168.86.21;
}

host k8s-worker2 {
  hardware ethernet 00:0c:29:5e:13:b7;
  fixed-address 192.168.86.22;
}

host k8s-worker3 {
  hardware ethernet 00:0c:29:c3:04:d3;
  fixed-address 192.168.86.23;
}
```

Start ISC DHCP server and enable it at startup.

```
systemctl enable isc-dhcp-server
systemctl start isc-dhcp-server
```

## Set up DNS server

Install bind and tools.

```
sudo apt-get install bind9 bind9utils bind9-doc
```

ACL, options and forwarding.

```
--> /etc/bind/named.conf.options

acl clients {
        192.168.0.0/16;
        localhost;
        localnets;
};

options {
        directory "/var/cache/bind";

        dnssec-enable yes;
        dnssec-validation yes;

        auth-nxdomain no;    # conform to RFC1035
        listen-on-v6 { any; };

        recursion yes;
        allow-query { clients; };

        forwarders {
                8.8.8.8;
                8.8.4.4;
        };
};
```

Local File with forward zone and reverse zone.

```
--> /etc/bind/named.conf.local

zone "brucejacobs.org" {
    type master;
    file "/etc/bind/zones/db.brucejacobs.org"; # zone file path
};

zone "86.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.192.168.86";  # 192.168.86.0/24 subnet
};
```

Forward zone file - define DNS records for forward DNS lookups (DNS name to IP).

```
--> /etc/bind/zones/db.brucejacobs.org
$TTL    604800
@       IN      SOA     networker.brucejacobs.org. admin.networker.brucejacobs.org. (
                  3       ; Serial
             604800     ; Refresh
              86400     ; Retry
            2419200     ; Expire
             604800 )   ; Negative Cache TTL
;
; name servers - NS records
     IN      NS      networker.brucejacobs.org.

; name servers - A records
networker.brucejacobs.org.          IN      A     192.168.86.15

; 10.98.95.0/24 - A records
k8s-controller1.brucejacobs.org.            IN      A      192.168.86.11
k8s-controller2.brucejacobs.org.            IN      A      192.168.86.12
k8s-controller3.brucejacobs.org.            IN      A      192.168.86.13
k8s-worker1.brucejacobs.org.                IN      A      192.168.86.21
k8s-worker2.brucejacobs.org.                IN      A      192.168.86.22
k8s-worker3.brucejacobs.org.                IN      A      192.168.86.23
k8s-api.brucejacobs.org.                    IN      A      192.168.86.18
media.brucejacobs.org.                      IN      A      192.168.86.16
```

Reverse zone file - define DNS PTR records for reverse DNS lookups (IP to DNS name).

```
--> /etc/bind/zones/db.192.168.86
$TTL    604800
@       IN      SOA     networker.brucejacobs.org. admin.networker.brucejacobs.org. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
; name servers
     IN      NS      networker.brucejacobs.org.

; PTR Records
15              IN      PTR     networker.brucejacobs.org.          ; 192.168.86.15
11              IN      PTR     k8s-controller1.brucejacobs.org.    ; 192.168.86.11
12              IN      PTR     k8s-controller2.brucejacobs.org.    ; 192.168.86.12
13              IN      PTR     k8s-controller3.brucejacobs.org.    ; 192.168.86.13
21              IN      PTR     k8s-worker1.brucejacobs.org.        ; 192.168.86.21
22              IN      PTR     k8s-worker2.brucejacobs.org.        ; 192.168.86.22
23              IN      PTR     k8s-worker3.brucejacobs.org.        ; 192.168.86.23
18              IN      PTR     k8s-api.brucejacobs.org.            ; 192.168.86.18
16              IN      PTR     media.brucejacobs.org.              ; 192.168.86.16
```

Start bind9 and enable it at startup.

```
systemctl enable bind9
systemctl start bind9
```

## Set up NFS server

View storage devices.

```
> lsblk 
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
loop0    7:0    0 86.6M  1 loop /snap/core/4486
sda      8:0    0    3G  0 disk 
├─sda1   8:1    0    1M  0 part 
└─sda2   8:2    0    3G  0 part /
sdb      8:16   0   40G  0 disk 
sr0     11:0    1 1024M  0 rom  
```

Confirm next partition number to use.

```
> sudo gdisk -l /dev/sda
GPT fdisk (gdisk) version 1.0.3

Problem opening /dev/sdab for reading! Error is 2.
The specified file does not exist!
defo@media:~$ sudo gdisk -l /dev/sda 
GPT fdisk (gdisk) version 1.0.3

Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: present

Found valid GPT with protective MBR; using GPT.
Disk /dev/sda: 6291456 sectors, 3.0 GiB
Model: Virtual disk    
Sector size (logical/physical): 512/512 bytes
Disk identifier (GUID): 304C050B-5B24-4A9E-AAEA-E53C45DDCFF0
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 6291422
Partitions will be aligned on 2048-sector boundaries
Total free space is 4029 sectors (2.0 MiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048            4095   1024.0 KiB  EF02  
   2            4096         6289407   3.0 GiB     8300  
```

Create partition on a disk dedicated for NFS sharing.

```
> sudo gdisk /dev/sdb
GPT fdisk (gdisk) version 1.0.3

Partition table scan:
  MBR: not present
  BSD: not present
  APM: not present
  GPT: not present

Creating new GPT entries.

Command (? for help): n
Partition number (1-128, default 1): 
First sector (34-83886046, default = 2048) or {+-}size{KMGTP}: 
Last sector (2048-83886046, default = 83886046) or {+-}size{KMGTP}: 
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): 8300
Changed type of partition to 'Linux filesystem'

Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): y
OK; writing new GUID partition table (GPT) to /dev/sdb.
The operation has completed successfully.
```

Create filesystem on a partition.

```
> sudo mkfs.ext4 -L nfs /dev/sdb1

mke2fs 1.44.1 (24-Mar-2018)
Discarding device blocks: failed - Remote I/O error
Creating filesystem with 10485499 4k blocks and 2621440 inodes
Filesystem UUID: 0968159a-720b-4a5c-9d76-1559e2acf2ad
Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
        4096000, 7962624

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (65536 blocks): done
Writing superblocks and filesystem accounting information: done
```

Mount partition on startup.

```
> sudo mkdir /mnt/nfs
> blkid
/dev/sda2: UUID="41de22aa-88eb-11e8-8ae2-000c293cce53" TYPE="ext4" PARTUUID="18878d4a-2df5-4d3c-9efc-c44bee6aee5d"
/dev/sdb1: LABEL="nfs" UUID="0968159a-720b-4a5c-9d76-1559e2acf2ad" TYPE="ext4" PARTLABEL="Linux filesystem" PARTUUID="4631e537-9db0-4079-8a44-382ac6fce7ef"

--> sudo vim /etc/fstab
UUID=41de22aa-88eb-11e8-8ae2-000c293cce53 / ext4 defaults 0 0
UUID=0968159a-720b-4a5c-9d76-1559e2acf2ad / ext4 defaults 0 1

```

Install and configure NFS.

```
sudo apt-get install -y nfs-kernel-server
```

Create a share for the first Persistent Volume.

```
> sudo mkdir /mnt/nfs/pv0001

--> /etc/exports
/mnt/nfs/pv0001        10.98.95.0/24(rw,root_squash,no_wdelay,no_subtree_check)
```

```
sudo systemctl start nfs-kernel-server
sudo systemctl enable nfs-kernel-server
```
