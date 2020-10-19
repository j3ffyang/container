
# Linux Kernel

#### Objective

I don't want to list all of them and just pick some of the most important parameters used in production Linux server.

#### Important Parameters

command | meaning | value
-- | -- | --
`uname -a` | kernel level | `Linux debian 4.19.0-10-amd64 #1 SMP Debian 4.19.132-1 (2020-07-24) x86_64 GNU/Linux`
`cat /proc/cmdline` | booting img | `BOOT_IMAGE=/vmlinuz-4.19.0-10-amd64 root=/dev/mapper/debian--vg-root ro quiet`
`sysctl -a | grep net.ipv4.ip_forward` | whether IP forwarding is enabled | `net.ipv4.ip_forward = 0`
`sysctl -a | grep shm` | This command displays the details of the shared memory segment sizes | `kernel.shmall = 18446744073692774399`<br>`kernel.shmmax = 18446744073692774399`<br>`kernel.shmmni = 4096`
`/sbin/sysctl -a | grep file-max` | This command displays the maximum number of file handles | `fs.file-max = 9223372036854775807`
`/sbin/sysctl -a | grep ip_local_port_range` | Display port range | `net.ipv4.ip_local_port_range = 32768	60999`
`cat /proc/sys/net/ipv4/conf/all/accept_redirects` | Accept ICMP redirect messages | `1`
`cat /proc/sys/net/ipv4/conf/all/mc_forwarding` | multi-casting forwarding | `0`
`cat /proc/sys/net/ipv4/conf/all/proxy_arp` | proxy arp | `0`
`cat /proc/sys/net/ipv4/conf/all/rp_filter` | enable source validation | `0`

> Reference >
[https://access.redhat.com/.../kernel_administration_guide/working_with_sysctl_and_kernel_tunables](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/kernel_administration_guide/working_with_sysctl_and_kernel_tunables)
[https://docs.oracle.com/en/database/oracle.../changing-kernel-parameter-values.html](https://docs.oracle.com/en/database/oracle/oracle-database/19/cwlin/changing-kernel-parameter-values.html)
https://datacadamia.com/os/linux/kernel_parameter_management



#### Shared Memory

```sh
jeff@debian:~$ ipcs -l

------ Messages Limits --------
max queues system wide = 32000
max size of message (bytes) = 8192
default max size of queue (bytes) = 16384

------ Shared Memory Limits --------
max number of segments = 4096
max seg size (kbytes) = 18014398509465599
max total shared memory (kbytes) = 18014398509481980
min seg size (bytes) = 1

------ Semaphore Limits --------
max number of arrays = 32000
max semaphores per array = 32000
max semaphores system wide = 1024000000
max ops per semop call = 500
semaphore max value = 32767
```

Beginning with the first section on Shared Memory Limits, the SHMMAX limit is the maximum size of a shared memory segment on a Linux system. The SHMALL limit is the maximum allocation of shared memory pages on a system.
It is recommended to set the SHMMAX value to be equal to the amount of physical memory on your system. However, the minimum required on x86 systems is 268435456 (256 MB) and for 64-bit systems, it is 1073741824 (1 GB).

> Reference >
https://www.ibm.com/support/knowledgecenter/en/SSEPGG_10.5.0/com.ibm.db2.luw.qb.server.doc/doc/t0008238.html
