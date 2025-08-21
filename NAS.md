쿠버네티스와 nas 연동: /etc/fstab에 이거 추가해야함. csp에도 설정 필요

 ```bash
 # /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/sda1 during curtin installation
/dev/disk/by-uuid/7e7db91c-db14-48b3-aa44-8f89d61114af / ext4 defaults 0 1
#/swap.img      none    swap    sw      0       0

10.242.11.242:/hm_ops_nas /nas nfs defaults,_netdev,vers=3 0 0
 ```
