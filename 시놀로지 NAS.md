맥북에서 NAS 접속방법: 커맨드 + K 누르고 smb://10.185.27.28, 이후 마운트할 볼륨 선택

권한을 관리할 폴더는 반드시 공유폴더로 생성해야 함

쿠버네티스와 nas 연동: /etc/fstab에 이거 추가해야함. csp에도 설정 필요

컴퓨터에서 폴더로 접근 가능하게 하려면 사용자 및 그룹 -> 응용 프로그램에서 SMB 켜야함. 이외에도 DSM, File Station 허용 필요

SMB에서 권한 없는 폴더가 보이지 않도록 하려면 제어판의 파일 서비스 탭에서 권한이 없는 사용자로부터 공유 폴더 숨기기

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
