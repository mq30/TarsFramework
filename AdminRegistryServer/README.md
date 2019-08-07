####tarsAdminRegistry管理主控

####配置项
配置项|描述
------------------------------------|----------------
`/tars/reap<updateHeartInterval>`   |服务心跳更新时间间隔，默认值10，最小值5，单位秒
`/tars/reap<registryTimeout>`       |管理主控心跳超时时间
`/tars/reap<heartbeatoff>`          |是否关闭更新管理主控心跳时间，值[Y|N]，一般需要迁移时，设置此项为Y
`/tars/objname<patchServerObj>`     |发布代理服务Obj，默认tars.tarspatch.PatchObj，建议按默认处理
`/tars/nodeinfo<ping_node_timeout>` |ping节点超时时间，默认3000，单位毫秒


&nbsp;

####功能项
功能项|描述
-------------------------------|----------------
注册管理主控                    |
定时更新管理主控心跳              |
定时加载物理分组信息到缓存        |
定时更新所有主控状态              |
任务管理                         |     
服务管理                         |                               


&nbsp;

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

&nbsp;

####管理命令及其流程
#####任务管理命令
命令|描述
-------------------------------|----------------
请求执行任务                      |
获取任务状态                      |
获取历史任务列表信息              |
更新任务状态                      |
任务管理                          | 

&nbsp;

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

&nbsp;

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

&nbsp;

#####应用服务管理流程
######卸载服务流程
``` 
删除服务DB记录信息，参考sql[4]。目前卸载服务并没有删除对应的安装包[TODO]
```
######获取应用名列表
查询DB，参考sql[9]

&nbsp;

#####节点管理命令
命令|描述
-------------------------------|----------------  
获取在线节点名列表             |
获取节点版本                   | 
ping节点                       |
停止节点                       |

&nbsp;

#####节点管理流程
######获取在线节点名列表
``` 
查询DB，参考sql[10]
```
######获取节点版本
``` 
查询DB，参考sql[11]
```

&nbsp;

#####发布管理命令
命令|描述
-------------------------------|----------------  
批量发布                        |
更新发布状态                    |
获取发布进度                     |

&nbsp;

#####配置模板管理命令
命令|描述
-------------------------------|----------------  
