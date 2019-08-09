####tarsAdminRegistry管理主控

####配置项
配置项|描述
------------------------------------|----------------
`/tars/reap<updateHeartInterval>`   |服务心跳更新时间间隔，默认值10，最小值5，单位秒
`/tars/reap<registryTimeout>`       |管理主控心跳超时时间
`/tars/reap<heartbeatoff>`          |是否关闭更新管理主控心跳时间，值[Y|N]，一般需要迁移时，设置此项为Y
`/tars/objname<patchServerObj>`     |发布代理服务Obj，默认tars.tarspatch.PatchObj，建议按默认处理
`/tars/nodeinfo<ping_node_timeout>` |ping节点超时时间，默认3000，单位毫秒
`/tars/nodeinfo<batchpatch_node_timeout>` |批量发布节点超时时间，默认10000，单位毫秒
`/tars/objname<AdminRegObjName>` |主控管理obj，默认tars.tarsAdminRegistry.AdminRegObj，建议按默认处理


####功能项
功能项|描述
-------------------------------|----------------
注册管理主控                    |
定时更新管理主控心跳              |
定时加载物理分组信息到缓存        |
定时更新所有主控状态              |
任务管理                         |     
服务管理                         |
节点代理缓存                      |
分组信息缓存                      |
发布管理                           |
节点管理                            |                               


####sql语句

1、注册管理主控/定时更新管理主控心跳
``` 
replace into t_registry_info(locator_id, servant, endpoint, last_heartbeat, present_state, tars_version) values('172.18.198.228:12000', 'tars.tarsAdminRegistry.AdminRegObj', 'tcp -h registry.tars.com -p 12000 -t 60000','2019-07-30 16:15:55','active','1.1.0_B002','2019-07-30 16:15:55','N');
```

2、定时加载物理分组信息到缓存
``` 
select group_id,ip_order,allow_ip_rule,denny_ip_rule,group_name from t_server_group_rule order by group_id;
```

3、定时更新所有主控（主控和管理主控）状态
``` 
update t_registry_info set present_state='inactive' where last_heartbeat < date_sub(now(), INTERVAL 150 SECOND);
``` 

4、卸载服务
``` 
delete from t_server_conf where application='' and server_name='' and node_name='';
delete from t_adapter_conf where application='' and server_name='' and node_name='';
```

5、请求执行任务
``` 
insert into t_task_item(task_no,item_no,application,server_name,node_name,command,parameters) values('','','','','','','','');
insert into t_task(task_no,serial,create_time,user_name) values('','','',now());
```

6、获取任务状态
``` 
select * from t_task as t1, t_task_item as t2 where t1.task_no=t2.task_no and t2.task_no='';
```

7、获取历史任务列表信息
``` 
select t1.create_time, t1.serial, t1.user_name, t2.* from t_task as t1, t_task_item as t2 where t1.task_no=t2.task_no and t2.application='' and t2.server_name='' and t2.command='' order by create_time desc, task_no;
```

8、更新任务状态
``` 
update t_task_item set start_time=now(),end_time=now(),status=2,log='' where item_no='';
```

9、获取应用名列表
``` 
select distinct application from t_server_conf;
```

10、获取在线节点名列表
``` 
select node_name, node_obj from t_node_info where present_state='active';
```

11、获取节点版本
``` 
select tars_version from t_node_info where node_name='';
``` 

12、查询节点obj
``` 
select node_obj from t_node_info where node_name='' and present_state='active'; 
```

13、获取服务列表
``` 
select application, server_name, node_name, setting_state, present_state,server_type from t_server_conf;
```

14、获取服务配置信息
``` 
select server.application, server.server_name, server.node_name, base_path,exe_path, setting_state, present_state, 
       adapter_name, thread_num, async_thread_num, endpoint,profile,template_name,allow_ip, max_connections, servant, 
       queuecap, queuetimeout,protocol,handlegroup,patch_version, patch_time, patch_user,server_type, start_script_path, 
       stop_script_path, monitor_script_path,config_center_port,enable_set, set_name, set_area, set_group
       from t_server_conf as server left join t_adapter_conf as adapter using(application, server_name, node_name)
       where server.node_name='' [and server.application='' and server.server_name='' and server.server_type != 'tars_dns'];
```

15、更新服务的设置状态
``` 
update t_server_conf set setting_state='' where application='' and server_name='' and node_name='';
```

16、更新服务的在线状态
``` 
update t_server_conf set present_state='' where application='' and server_name='' and node_name='';
```

17、获取发布文件信息
``` 
select tgz, md5 from t_server_patchs where id=0;
```

18、更新发布状态
``` 
update t_server_patchs set publish='1',publish_user='',publish_time=now(),lastuser='' where id=0;
```

19、获取模板内容
``` 
select template_name, parents_name, profile from t_profile_template where template_name='';
```

####管理命令及其流程

#####任务管理命令
命令|描述
-------------------------------|----------------
请求执行任务                      |
获取任务状态                      |
获取历史任务列表信息              |
更新任务状态                      |
任务管理                          | 

#####任务流程

######请求执行任务
```
保存任务信息到DB，参考sql[5]，成功保存到DB后，将任务放置入任务队列，等待执行
```
######获取任务状态
```
优先查询任务队列中是否已经有响应信息，若无则查询DB，参考sql[6]
```
#####获取历史任务列表信息
```
查询DB，参考sql[7]
```
#####更新任务状态
```
更新DB，参考sql[8]
```

#####任务流程图
[TODO]


#####应用服务管理命令
命令|描述
-------------------------------|----------------  
卸载服务                       |
获取应用名列表                 | 
获取服务列表                   |
获取特定服务状态               |
获取特定ip所属group            |
启动服务                        |
停止服务                        |
重启服务                        |
通知服务                        |
加载服务信息                    |

#####应用服务管理流程

######卸载服务流程
``` 
删除服务DB记录信息，参考sql[4]。目前卸载服务并没有删除对应的安装包[TODO]
```

######获取应用名列表
```
查询DB，参考sql[9]
```

######获取服务列表
```
查询DB，参考sql[13]
```

######获取特定服务状态
```
查询DB获取指定服务列表（异常情况可能保存了多条记录），只取第一条记录，参考sql[14]，然后获取节点代理，参考sql[12]，DB
主要查询两个状态：settingStateInReg和presentStateInReg。然后调用节点接口async_getStateInfo（异步处理），异常情况下，
同步调用接口getState和getServerPid，从节点获取到服务状态presentStateInNode和服务进程id，
具体错误码可参考AdminReg.tars或者其自动生成的AdminReg.h文件
```

######获取特定ip所属group
```
优先从服务分组缓存信息查找，查不着不到则从服务分组规则缓存中校验是否匹配模式（支持完全匹配和模糊匹配）,
其中服务分组规则由
```

######启动服务
```
更新DB服务的设置状态（active），参考sql[15]，获取节点代理，参考sql[12]，调用节点的startServer接口
```

######停止服务
```
更新DB服务的设置状态（inactive），参考sql[15]，获取节点代理，参考sql[12]，调用节点的stopServer接口
```

######重启服务
```
获取节点代理，参考sql[12]，调用节点的stopServer接口；更新DB服务的设置状态（active），参考sql[15]，获取节点代理，
参考sql[12]，调用节点的startServer接口
```

######通知服务
```
获取节点代理，参考sql[12]，调用节点的notifyServer接口
```

######加载服务信息
```
获取节点代理，参考sql[12]，调用节点的loadServer接口
```

#####节点管理命令
命令|描述
-------------------------------|----------------  
获取在线节点名列表             |
获取节点版本                   | 
ping节点                       |
停止节点                       |

#####节点管理流程

######获取在线节点名列表
``` 
查询DB，参考sql[10]
```

######获取节点版本
``` 
查询DB，参考sql[11]
```

######ping节点
```
根据节点obj查询节点代理缓存获取节点代，获取不到则查询DB获取节点obj，参考sql[12]，根据节点obj查询节点代理缓存获取节点代理，最后调用节点的tars_ping接口
```

######停止节点
```
根据节点obj查询节点代理缓存获取节点代，获取不到则查询DB获取节点obj，参考sql[12]，根据节点obj查询节点代理缓存获取节点代理，最后调用节点的shutdown接口
```

#####发布管理命令
命令|描述
-------------------------------|----------------  
批量发布                        |
更新发布状态                    |
获取发布进度                    |
预发布                          |[TODO]
灰度发布                        |[TODO]

#####发布管理流程

######批量发布
``` 
查询DB获取发布文件名和checksum，参考sql[17]，调用发布代理的preparePatchFile接口让tarspatch准备发布文件，
调用节点代理的async_patchPro进行发布
```

######更新发布状态
``` 
更新DB，参考sql[18]
```

######获取发布进度
``` 
调用节点代理的async_getPatchPercent获取发布进度
```

#####配置模板管理命令
命令|描述
-------------------------------|----------------  
获取模板信息                   |
获取服务模板信息               |

#####配置模板管理流程

######获取模板信息
``` 
查询DB，参考sql[19]
```

######获取服务模板信息
``` 
查询DB，参考sql[14]
```

