---
title: NFS
id: 27d131aa-1140-4e07-a57a-458c5c32d96d
date: 2024-03-29 23:46:37
auther: xmy
cover: null
excerpt: Network File System Sun公司开发， LPC：local procedure call 本地过程调用 RPC：remote procedure call 远程过程调用 编程技术，简化分布式应用开发 Portmap：监听在111/tcp 111/udp端ࡣ
permalink: /archives/nfs
categories:
 - linux
tags: 
 - linux
---

## Network File System    
Sun公司开发，  
* LPC：local procedure call   本地过程调用
* RPC：remote procedure call  远程过程调用  编程技术，简化分布式应用开发
* Portmap：监听在111/tcp 111/udp端口


yum -y install nfs-utils
主要进程/usr/sbin/rpc.nfsd、rpc.mountd

/etc/init.d/nfsd start  
nfsd nfs服务，一般监听在2049  
mountd 挂载服务，动态端口  
quotad 磁盘配额，动态端口  
/etc/init.d/rpcbind status 监听在111端口  

rpcinfo -p localhost    查看本机nfs监听端口  
showmount  列出本机上共享的   
showmount -e NFS_SERVER：查看NFS服务器“导出”的各文件系统  
showmount -a NFS_SERVER：查看NFS服务器所有被挂载的文件系统及客户端对应列表

/etc/exports
挂载远程NFS文件系统至本地：mount -t nfs NFS_SERVER:/PATH/TO/SOME_EXPORT       /PATH/TO/SOMEWHERE  
文件权限类型（使用man exports查看帮助文档）：  
    ro：只读   
    rw：读写  
    sync：同步  
    async：异步  
    root_squash：将root用户映射为来宾账户  
    no_root_squash： 
    all_squash： 
    anonuid，anongid：指定映射来宾用户的UID、GID  
    
exports -ra 重新输出映射共享  

可以在/etc/fsttab中配置挂载，使得开机自动挂载远程共享目录  
/etc/sysconfig/nfs   配置监听端口  


作业：通过NFS共享两套LAMP架构的web目录，实现目录共享  

