## Tarea 1
* Configurar ~/.vimrc
```
vim ~/.vimrc
filetype plug indent on
syntax on
au filetype yaml setlocal expandtab ts=2 sw=2 ai
set cursorline
set cursorcolumn
set nocompatible
hightlight cursorsolumn ctermbg=blue guibg=#1e90ff
```
* Instalar ansible
```
yum install ansible-core -y
dnf install ansible-core
dnf install rhel-system-roles
```
* Crear rutas de trabajo
```
mkdir -p /home/admin/ansible/roles /home/admin/ansible/mycollection
```
* ansible-cfg
```
ansible-config init --disabled -t all > ansible.cfg
```
```
[defaults]
remote_user =
inventory = /home/admin/ansible/inventory
roles_path = /home/admin/ansible/roles:/usr/share/ansible/roles:~/.ansible/roles:/etc/ansible/roles
collections_paths = /home/admin/ansible/mycollection:/usr/share/ansible/collections:/root/.ansible/collections
host_key_checking = false

[privilege-escalations]
become = true
become_method = sudo
become_user = root
```
* inventory
```
[dev]
node1 ansible_host=192.168.0.111
[test]
node2 ansible_host=192.168.0.112
[webservers:children]
dev
```
* Export variable
```
export ANSIBLE_CONFIG=/home/admin/ansible/ansible.cfg
```

======
## Tarea 2 - Crear archivo yum.yml y repositorios
```
ansible-doc -l 
ansible-doc rpm_key
ansible-doc yum_repository
```
```
yum.yml
---
- name: Configurar repositorios
  hosts: all
  tasks:
    - name: Repo EX294_BASE
      yum_repository:
        name: EX294_BASE
        description: EX294 base software
        baseurl: http://
        gpgcheck: yes
        gpgkey: http://
        enabled: yes
    - name: Repo EX294_STREAM
      yum_repository:
        name: EX294_STREAM
        description: EX294 stream software
        baseurl: http://
        gpgcheck: yes
        gpgkey: http://
        enabled: yes
```
```
ansible all -a "ls /etc/yum.repos.d"
```

======
## Tarea 3 - Install packages packages.yml
```
ansible-doc yum
```
packages.yml
```
---
- name: packages installation
  hosts:
    - dev
    - test
    - prod
  tasks:
    - name: Install php y mariadb-server
      yum:
        name:
          - php
          - mariadb-server
        state: present

    - name: Install RPM on dev
      when: inventory_hostname in groups['dev']
      yum:
      name: "@RPM Development tools" 
      state: latest

    - name: Install lastest packages on dev
      when: inventory_hostname in groups['dev']
      yum:
      name: [*]
      state: latest        
```
=====
## Tarea 4 - Content collections
```
ansible-doc -t collection redhat.rhel_system_roles
ansible-galaxy collection install XXX -p /home/admin/ansible/mycollection
ansible-galaxy collection list -p /home/admin/ansible/mycollection
```
=====
## Tarea 5 - Install roles using ansible-galaxy (balance/phpinfo)

requirements.yml
```
- src: http://
  name: balancer
- src: http://
  name: phpinfo
```
```
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install -r 
ansible-galaxy list
ansible-galaxy init balancer/phpinfo
```

=====
## Tarea 6 - Timesync
```
yum install rhel-system-roles
ansible-galaxy list
vim /usr/share/ansible/roles/rhel-system-roles.timesync/README.md
```
timesync.yml
```
---
- name: task 6
  hosts: all
  vars:
    timesync_ntp_servers:
      - hostname: 172.24.1.254
        iburst: yes
  roles:
    - rhel-system-roles.timesync
```
```
timedatectl
grep 172 /etc/chrony.conf
```

=====
## Tarea 7 - Crear role y usarlo

* Crear role
```
ansible-galaxy init apache
```
roles/apache/tasks/main.yml
```
---
- name: Install http
  yum:
    name: httpd
    state: present
- name: Start service
  service:
    name: httpd
    state: started
    enabled: yes
- name: Start firewalld service
  service: 
    name: firewalld
    state: started
    enabled: yes
- name: Add http service in fiewall rule
  firewalld:
    service: http
    state: enabled
    permanent: yes
    immediante: yes
- name: Copy template.j2 to server directory
  template:
    src: index.html.j2
    dest: /var/www/html/index.html
```
* Crear index apache/template/index.html.j2
```
Welcome to {{ ansible_fqdn }} on {{ ansible_default_ipv4.address }}
```
* Crear ansible/newrole.yml
```
- hosts: webservers
  roles:
    - apache
```

=====
## Tarea 8 - Balancer
* Crear roles para el ejercicio
ansible/roles.yml
```
- name: Role haproxy on balancer group
  roles:
    - balancer
- name: Role phpinfo on webservers group
  roles:
    - phpinfo
```
### Preparación Balancer Role
roles/balancer/tasks/main.yml
```
- name: Install haproxy
  yum:
    name: haproxy
    state: present
- name: Copy haproxy configuration
  template:
    src: haproxy.cfg.j2
    dest: /etc/haproxy/haproxy.cfg
- name: Enabled/started haproxy
  service: 
    name: haproxy
    state: started
    enabled: yes
- name: Permit http on firewalld
  firewalld:
    service: http
    permanent: yes
    immediate: yes
    state: enabled
```
roles/balancer/vars/main.yml
```
firewall_rule:
  - port: 80/tcp
haproxy_servers:
  - name: node3
    ip: 192
    backend_port: 80
  - name: node4
    ip: 192
    backend_port: 80
```
roles/balancer/templates/haproxy.cfg.j2
```
global
  daemon
defaults
  mode http
frontend http_fronend
  bind: *:80
  default_backend webservers
backend webservers
  balance roundrobin
  server node3 192:80 check
  server node4 192:80 check
```
### Preparación phpinfo Role
/roles/phpinfo/vars/main.yml
```
web_message: Hello PHP World
```
/roles/phpinfo/templates/hello.j2
```
<?php
echo " {{ web_message }} from {{ ansible_fqdn }} \n"
?>
```

=====
## Tarea 9 - Generate host file
ansible/hosts.j2
```
{% for host in groups['all'] %}
{{ hostvars[host].ansible_host }} {{ hostvars[host].ansible_fqdn | default(host) }} {{ host }}
{% endfor %}
```
ansible/host.yml
```
- name: task 9
  hosts: all
  tasks:
  - mane: copy /etc/hosts to /etc/myhosts
    when: inventory_hostname in groups['dev']
    template:
      src: /home/admin/ansible/hosts.j2
      dest: /etc/myhosts
```
=====
## Tarea 10 - Content issue
