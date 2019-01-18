## Centos 7 如何卸载 docker或者使用yum安装的软件
1. 首先搜索已经安装的docker 安装包 
   `yum list installed|grep docker`
   或者使用该命令 
   `rpm -qa|grep docker`

2. 分别删除安装包 
   `yum –y remove docker.x86_64 `
3.  删除docker 镜像 
   ` rm -rf /var/lib/docker `
4. 再次check docker是否已经卸载成功 
   `rpm -qa|grep docker` 
   如果没有搜索到，那么表示已经卸载成功。