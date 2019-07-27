## node_exporter

Node_exporter部署请参考 第二章 部署/二进制部署 中的Node_exporter。
还可以使用Ansible进行部署。

## Ansible Playbook部署node_exporter
node_exporter为golang开发编译的程序，直接运行即可。

### 1.新建ansible playbook roles
```
# 进入ansible roles目录
cd /etc/ansible/roles

# 初始化
ansible-galaxy init deploy_node_exporter

# 下载node_exporter
cd deploy_node_exporter/files
wget https://github.com/prometheus/node_exporter/releases/download/v0.18.1/node_exporter-0.18.1.linux-amd64.tar.gz

# 目录结构
tree deploy_node_exporter/
deploy_node_exporter/
├── defaults
│   └── main.yml
├── files
│   └── node_exporter-0.18.1.linux-amd64.tar.gz
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── README.md
├── tasks
│   └── main.yml
├── templates
├── tests
│   ├── inventory
│   └── test.yml
└── vars
    └── main.yml
```

### 2.编写playbook tasks
vi /etc/ansible/roles/deploy_node_exporter/tasks/main.yml
```
---
- name: create directory /ops
  file: 
    path: /ops
    state: directory
    mode: '0755'

- name: unarchive node_exporter
  unarchive:
    src: node_exporter-0.18.1.linux-amd64.tar.gz
    dest: /ops/

- name: configure node_exporter.service
  lineinfile:
    path: /usr/lib/systemd/system/node_exporter.service
    line: "{{ item }}"
    create: yes
    state: present
  with_items:
    - "[Unit]"
    - "Description=Node Exporter For Prometheus"
    - "[Service]"
    - "Type=simple"
    - "ExecStart=/ops/node_exporter-0.18.1.linux-amd64/node_exporter"
    - "[Install]"
    - "WantedBy=mutil-user.target"

- name: start and enable service
  shell: |
    systemctl daemon-reload 
    systemctl enable node_exporter
    systemctl start node_exporter
- name: add firewall port 9100
  shell: firewall-cmd --add-port=9100/tcp --permanent && firewall-cmd --reload
```

### 3.编写playbook
vi /etc/ansible/workflow/deploy_node_exporter.yml
其中hosts为需要安装的机器。
```
---
- hosts: node01
  remote_user: root
  roles:
    - deploy_node_exporter
```

### 4.部署node_exporter
```
ansible-playbook deploy_node_exporter.yml
```

### 5.验证
```
curl http://node01:9100/metrics
```
