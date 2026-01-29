当你已经处于内网 该如何搜集内网的主机

如果是在Linux ip a / ifconfig

namp 扫描存活主机 只探测是否存活 不扫描端口 nmap -sn 192.168.1.0/24

如果ICMP被禁 不依赖ICMP 基本无法被防火墙拦截 使用TCP/ARP 探测 arp-scan -l 或者 arp-scan 192.168.1/24

TCP 探测存活 nmap -Pn -p 80,443,445,22 192.168.1.0/24

寻找 路由里面的网段 Linux 用ip route 如果是在windows route print

fping -a -g 192.168.1.0/24 2 >dev/null -a 只显示存活 -g 生成ip范围
