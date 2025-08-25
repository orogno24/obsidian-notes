맥북에서 NAS 접속방법: 커맨드 + K 누르고 smb://10.185.27.28, 이후 마운트할 볼륨 선택

권한을 관리할 폴더는 반드시 공유폴더로 생성해야 함

쿠버네티스와 nas 연동: /etc/fstab에 이거 추가해야함. csp에도 설정 필요

컴퓨터에서 폴더로 접근 가능하게 하려면 사용자 및 그룹 -> 응용 프로그램에서 SMB 켜야함. 이외에도 DSM, File Station 허용 필요

SMB에서 권한 없는 폴더가 보이지 않도록 하려면 제어판의 파일 서비스 탭에서 권한이 없는 사용자로부터 공유 폴더 숨기기


![[스크린샷 2025-08-25 오후 5.50.41.png]]
폐쇄망 NAS 구축 방법: NAS에 폐쇄망 랜선 연결하면 네트워크 인터페이스에 IP 출력됨. 이후 해당 IP를 기반으로 각 PC의 IP 세팅 필요(인터넷망, 폐쇄망 둘다 연결한 예시)

PC에 등록한 NAS 정보 삭제하는 방법: cmd 열어서 net use \\\10.185.27.32 /delete 명령어 입력하고 재부팅

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
