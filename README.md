Filebeat role
=========

Роль для установки Filebeat на хостах с ОС: Debian, Ubuntu, CentOS, RHEL.

Requirements
------------

Поддерживаются только ОС семейств debian и EL.

Role Variables
--------------

| Variable name | Default | Description |
|-----------------------|----------|-------------------------|
| filebeat_version | "7.14.0" | Параметр, который определяет какой версии Filebeat будет установлен |

Example Playbook
----------------

    - hosts: all
      roles:
         - { role: filebeat_roles }

License
-------

BSD

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
- Zakharov Sergey, Netology student of DevOps engeneers course.
- Files version 0.0.01
---
### Описание функционала `ansible-playbook` согласно ДЗ.
* Показана структура файлов роли с описанием задач, модулей, методов, переменных и их значений, при выполнении которых Ansible на управляемых нодах установит утилиты и настроит соответствующие конфигурации.

1. Файл ` filebeat_roles/defaults/main.yml `
```yml
elk_stack_version: "7.14.0"
```
2. Файл `
filebeat_roles/handlers/main.yml `
```yml
- name: restart filebeat
  become: true
  service:
    name: filebeat
    state: restarted
```
3. Файл ` filebeat_roles/tasks/main.yml `
```yml
- include_tasks: "download_filebeat_rpm.yml"
- include_tasks: "install_filebeat.yml"
- import_tasks: "configure_filebeat.yml"
- include_tasks: "set_filebeat_systemwork.yml"
- include_tasks: "load_kibana_dashboard.yml"
```
4. Файл ` filebeat_roles/tasks/download_filebeat_rpm.yml `
```yml
- name: "Download Filebeat's rpm"
  get_url:
    url: "https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-{{ elk_stack_version }}-x86_64.rpm"
    dest: "/tmp/filebeat-{{ elk_stack_version }}-x86_64.rpm"
  register: download_filebeat
  until: download_filebeat is succeeded
```
5. Файл ` filebeat_roles/tasks/install_filebeat.yml `
```yml
- name: Install Filebeat
  become: true
  yum:
    name: "/tmp/filebeat-{{ elk_stack_version }}-x86_64.rpm"
    state: present
  notify: restart filebeat
```
6. Файл ` filebeat_roles/tasks/configure_filebeat.yml `
```yml
- name: Configure Filebeat
  become: true
  template:
    src: filebeat.yml.j2
    mode: 0644
    dest: /etc/filebeat/filebeat.yml
  notify: restart filebeat
```
7. Файл ` filebeat_roles/tasks/set_filebeat_systemwork.yml `
```yml
- name: Set filebeat systemwork
  become: true
  command:
    cmd: filebeat modules enable system
    chdir: /usr/share/filebeat/bin
  register: filebeat_modules
  changed_when: filebeat_modules.stdout != 'Module system is alredy enabled'
```
8. Файл ` filebeat_roles/tasks/load_kibana_dashboard.yml `
```yml
- name: Load Kibana dashboard
  become: true
  command:
    cmd: filebeat setup
    chdir: /usr/share/filebeat/bin
  register: filebeat_setup
  changed_when: false
  until: filebeat_setup is succeeded
```
9. Файл ` filebeat_roles/templates/filebeat.yml.j2 `
```yml
output.elasticsearch:
  hosts: ["http://{{ hostvars['el-instance']['ansible_facts']['default_ipv4']['address'] }}:9200"]
setup.kibana: 
  host: "http://{{ hostvars['k-instance']['ansible_facts']['default_ipv4']['address'] }}:5601"
filebeat.config.modules.path: ${path.config}/modules.d/*.yml
```
10. Файл ` filebeat_roles/vars/main.yml `
```yml
filebeat_version: "7.14.0"
```
