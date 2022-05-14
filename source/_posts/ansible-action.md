# Ansible playbook 批量部署 node-exporters 服务


title: Ansible playbook 批量部署 node-exporters 服务
date: 2020-08-11 22:59:30
tags: [Ansible] 
categories: [技术]

------




假如现在有多台机器`10.1.2.3,10.1.2.4,10.1.2.5` 需要同时部署node exporter.

单个弄挺麻烦的，而且可能有加机器的需求，最好的办法是写好模板，改下IP重新运行就能部署上去了

之前有尝试[ansible](https://counter2015.com/2020/06/12/ansible), 正好可以用上

按照上文提到的方式，ansible的配置文件地址在`/etc/ansible/hosts`

配置文件支持`ini`格式或者`yaml`格式，这里使用的是`yaml`

```yaml
all:
  children:
	  node_exporters:
	    hosts:
	      10.1.2.3:
	      10.1.2.4:
	      10.1.2.5:
```

对于连续的ip地址，可以用范围来表示

```yaml
all:
  children:
    node_exporters:
      hosts:
        10.1.2.[3-5]:
```

详细的有关范围相关的配置可以参见[官方文档](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#adding-ranges-of-hosts)

配置完成后，需要设置账户机器之间的互信

多台机器的批量验证可以参考如下脚本

```shell
#!/bin/bash
#
# generate ssh key on deploy node, copy ssh-key to remote node
# ./ssh_no_password.sh $user_pwd $ip_list
#
CUR_DIR=$(cd `dirname $0`; pwd)

if [ $# -lt 2 ]; then
    echo "FATAL para not enough"
    exit 1
fi

USER_PWD=$1
IP_LIST=$2

# 如果不存在签名文件，生成一份
if [ ! -f /home/$USER/.ssh/id_rsa ]; then
    expect -c "
        spawn ssh-keygen -t rsa
            expect {
                \"*key*\" {send \"\r\"; exp_continue}     
                \"*passphrase*\" {send \"\r\"; exp_continue}     
                \"*again*\" {send \"\r\";}  
            }
    "
fi

# 用 expect 来实现自动填写密码
for IP in $(cat $IP_LIST)
do 
    expect -c "
        spawn ssh-copy-id -i /home/$USER/.ssh/id_rsa $USER@$IP
            expect {
                \"*yes/no*\" {send \"yes\r\"; exp_continue}     
                \"*password*\" {send \"$USER_PWD\r\"; exp_continue}     
                \"*Password*\" {send \"$USER_PWD\r\";}
            }
    "
done    

for IP in $(cat $IP_LIST)
do 
    expect -c "
        spawn ssh $USER@$IP "exit"
            expect {
                \"*yes/no*\" {send \"yes\r\"; exp_continue}
            }
    "
done 
```





完成后，可以使用如下方法验证

```shell
$ ansible node_exporters -m ping
10.1.2.3 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}
10.1.2.4 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}
10.1.2.5 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}
```







ansible playbook的使用方法，[官方文档](https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html)讲的很仔细，这里就不再赘述了

上面的验证用playbook来写就是这样

```yaml
# ping.yml
- hosts: node_exporters
  remote_user: root
  tasks:
    - name: test connection
      ping:
      remote_user: root
```



测试如下

```bash
$ ansible-playbook ping.yml

PLAY [node_exporters] ***************************************************************************************************************

TASK [Gathering Facts] **************************************************************************************************************
ok: [10.1.2.3]
ok: [10.1.2.4]
ok: [10.1.2.5]

TASK [test connection] **************************************************************************************************************
ok: [10.1.2.3]
ok: [10.1.2.4]
ok: [10.1.2.5]

PLAY RECAP **************************************************************************************************************************
10.1.2.3                 : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
10.1.2.4                 : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
10.1.2.5                 : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```



为了保证重新执行playbook是安全的，需要先对每一步操作的状态做校验，以保证整体操作的幂等性

在[jinja2](https://jinja.palletsprojects.com/en/2.11.x/)语法外面加引号，是因为 [yaml gotcha](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#hey-wait-a-yaml-gotcha)

```yaml
- hosts: node_exporters
  remote_user: root
  vars:
    node_exporter: node_exporter-0.18.1.linux-amd64
    dest_dir: /tmp/4t
    deploy_user: your_user_name
    node_port: 9100
  tasks:
    - name: test connection
      ping:
      
    - name: create directory if they dont exist
      file:
        path: "{{ dest_dir }}"
        state: directory
        mode: 0775
        
    - name: extract files
      unarchive:
        src: "/home/{{ deploy_user }}/installs/{{ node_exporter }}.tar.gz"
        dest: "{{ dest_dir }}"
        force: no
        mode: 0755
      
    - name: start node_exporter
      shell: "nohup {{ dest_dir }}/{{ node_exporter }}/node_exporter --web.listen-address=':{{ node_port }}' &"
        
    - wait_for: 
        host: 127.0.0.1 
        port: "{{ node_port }}" 
        timeout: 1
    - debug: msg=ok
```



运行结果如下

```bash
$ ansible-playbook node.yml 

PLAY [node_exporters] ***************************************************************************************************************

TASK [Gathering Facts] **************************************************************************************************************
ok: [10.1.2.3]
ok: [10.1.2.4]
ok: [10.1.2.5]

TASK [test connection] **************************************************************************************************************
ok: [10.1.2.3]
ok: [10.1.2.4]
ok: [10.1.2.5]

TASK [create directory if they dont exist] ******************************************************************************************
ok: [10.1.2.3]
ok: [10.1.2.4]
ok: [10.1.2.5]

TASK [extract files] ****************************************************************************************************************
ok: [10.1.2.3]
ok: [10.1.2.4]
ok: [10.1.2.5]

TASK [start node_exporter] **********************************************************************************************************
changed: [10.1.2.3]
changed: [10.1.2.4]
changed: [10.1.2.5]

TASK [wait_for] *********************************************************************************************************************
ok: [10.1.2.3]
ok: [10.1.2.4]
ok: [10.1.2.5]

TASK [debug] ************************************************************************************************************************
ok: [10.1.2.3] => {
    "msg": "ok"
}
ok: [10.1.2.4] => {
    "msg": "ok"
}
ok: [10.1.2.5] => {
    "msg": "ok"
}

PLAY RECAP **************************************************************************************************************************
10.1.2.3                 : ok=7    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
10.1.2.4                 : ok=7    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
10.1.2.5                 : ok=7    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```


