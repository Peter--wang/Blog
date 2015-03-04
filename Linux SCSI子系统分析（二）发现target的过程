[TOC]
Linux SCSI子系统分析（二）发现target的过程
=
发现过程
-
发现target的过程比较简单，从复杂的代码中总结出主要的流程如下图。这个流程和命令iscsiadm -m discovery -t st -p 192.168.8.1相对应：
```sequence
iscsiadm->iscsid:取得默认iscsi配置
iscsiadm->tgtd:创建会话
iscsiadm->iscsiadm:创建链接
iscsiadm->tgtd:握手
tgtd-->iscsiadm:发送所有target
iscsiadm->iscsiadm:输出所有target
```
这个发现的流程的代码写的挺复杂，但是总结出的流程还是比较简单的。主要就是iscsiadm这个程序和tgtd这个程序通过socket交互，取得所有的target。从tgtd侧返回的target格式如下：
  > "TargetName=iqn.2013-07.sds.ginkgo:ws<font color="red">\0</font>TargetAddress=192.168.8.1:3260,1<font color="red">\0</font>"
   
注意看加红的地方，每个属性都是以字符休止符隔开的
函数调用过程
-
    经过下图的函数调用流程，initiator端就取得到了target端所有的iqn
```sequence
iscsid->iscsid:main
iscsid->iscsid:mgmt_ipc_listen
tgtd->tgtd:main
tgtd->tgtd:iscsi_portal_create\n创建target成功后\n会注册相应的回调函数\n用于处理target对应的请求
tgtd->tgtd:event_loop
iscsiadm->iscsiadm:main
iscsiadm->iscsiadm:exec_dsc_op
iscsiadm->iscsiadm:do_sendtargets
iscsiadm->iscsiadm:idbm_sendtargets_defaults
iscsiadm->iscsiadm:idbm_sync_config
iscsiadm->iscsid:socket
iscsid->iscsid:mgmt_ipc_handle\nIPC处理函数\n用于处理IPC请求
iscsid-->iscsiadm:config
iscsiadm->iscsiadm:do_software_sendtargets
iscsiadm->iscsiadm:discovery_sendtargets
iscsiadm->iscsiadm:iscsi_alloc_session
iscsiadm->iscsiadm:request_initiator_name
iscsiadm->iscsid:socket
iscsid->iscsid:mgmt_ioc_handle
iscsid->iscsiadm:initiator config
iscsiadm->iscsiadm:iscsi_create_session
iscsiadm->iscsiadm:iscsi_login
iscsiadm->tgtd:socket
tgtd->tgtd:iscsi_tcp_event_handle
tgtd->tgtd:cmnd_execute
tgtd->tgtd:cmnd_exec_login
tgtd->iscsiadm:succeed
iscsiadm->iscsiadm:request_targets
tgtd->tgtd:cmnd_exec_text
tgtd->tgtd:text_scan_text
tgtd->tgtd:target_list_build
tgtd->iscsiadm:"TargetName=iqn.2013-07.sds.ginkgo:ws\0\nTargetAddress=192.168.8.1:3260,1\0
```    
