### Описание функционала `Filebeat role` согласно ДЗ.

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

* Показана структура файлов роли с описанием задач, модулей, методов, переменных и их значений, при выполнении которых Ansible на управляемых нодах установит утилиты и настроит соответствующие конфигурации.

1. Файл ` filebeat_roles/defaults/main.yml ` задает версию Kibana. Эта версия должна быть аналогична версии Elasticsearch
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
* `- name: restart filebeat`   Имя первого хендлера 
* `become: true`      Выполнять с повышением привелегий (под УЗ root по дефолту)
* `service:`      Указание на то, что нужно обратиться к модулю `ansible.builtin.service`
* `name: filebeat`   Имя сервиса
* `state: restarted`      Выполнить команду рестарта

3. Файл ` filebeat_roles/tasks/main.yml `задает порядок выполнения задач в роли.
```yml
- include_tasks: "download_filebeat_rpm.yml"
- include_tasks: "install_filebeat.yml"
- import_tasks: "configure_filebeat.yml"
- include_tasks: "set_filebeat_systemwork.yml"
- include_tasks: "load_kibana_dashboard.yml"
```
* `include_tasks: "download_filebeat_rpm.yml"` Запуск задачи загрузки Filebeat
* `include_tasks: "install_filebeat.yml"`       Запуск задачи инсталляции Filebeat
* `import_tasks: "configure_filebeat.yml"`      Запуск задачи конфигурирования Filebeat
* `include_tasks: "set_filebeat_systemwork.yml"`        Запуск задачи старта сбора логов системы для передачи их в Elasticsearch
* `include_tasks: "load_kibana_dashboard.yml"`      Запуск задачи загрузки дашборда Кибаны


4. Файл ` filebeat_roles/tasks/download_filebeat_rpm.yml `
```yml
- name: "Download Filebeat's rpm"
  get_url:
    url: "https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-{{ elk_stack_version }}-x86_64.rpm"
    dest: "/tmp/filebeat-{{ elk_stack_version }}-x86_64.rpm"
  register: download_filebeat
  until: download_filebeat is succeeded
```
* `- name: "Download Filebeat's rpm"`  Название задачи для понимания ее назначения
* `get_url:`    Ипользовать модуль `get_url`
* ` url: "https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-{{ elk_stack_version }}-x86_64.rpm"`    Адрес ссылки для загрузки `Filebeat`. Переменая `{{ elk_stack_version }}` указывает на то, какую версию нужно загружать.
* `dest: "/tmp/filebeat-{{ elk_stack_version }}-x86_64.rpm"`  Директория на `manage_node` для сохранения загруженного пакета 
* `register: download_filebeat`      В переменную `download_filebeat` записываем результат выполнения загрузки.
* `until: download_filebeat is succeeded`    Цикл для выполнения условий успешной загрузки пакета.

5. Файл ` filebeat_roles/tasks/install_filebeat.yml `
```yml
- name: Install Filebeat
  become: true
  yum:
    name: "/tmp/filebeat-{{ elk_stack_version }}-x86_64.rpm"
    state: present
  notify: restart filebeat
```
* `- name: Install Filebeat`      Имя задачи.
* `become: true`      Выполнять с повышением привелегий (под УЗ root по дефолту)
* `yum:`    Запустить модуль `yum` для установки в систему утилиты из пакета RPM
* `name: "/tmp/filebeat-{{ elk_stack_version }}-x86_64.rpm"`   Место расположения пакета.
* `state: present`    Условие гарантии того, что данный пакет будет установлен.
* `notify: restart filebeat`   Выполнить условие - с помощью хендлера с именем `restart filebeat`  перезапустить сервис  `Filebeat` на `manage_node`

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
* `- name: Configure Filebeat`   Имя задачи.
* `become: true`      Выполнять с повышением привелегий (под УЗ root по дефолту)
* `template:`     Обращение к модулю `template` для создания на `manage_node` из файлов с шаблоном конфигурации .j2 файлов конфигураци .yml. Модуль ищет шаблоны  в директории `/template` на `control_node`
* `src: filebeat.yml.j2`   Указан файл - источник шаблона
* `dest: /etc/filebeat/filebeat.yml`    Указано место назначения файла конфигурации.
* `notify: restart filebeat`   Выполнить условие - с помощью хендлера с именем `restart filebeat`  перезапустить сервис  `filebeat` на `manage_node`

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
* `- name: Set filebeat systemwork`     Имя задачи
* `become: true`      Выполнять с повышением привелегий (под УЗ root по дефолту)
* `command:`      Использование модуля `command`. Этот модуль выполняет системные команды на `manage_node`
* `cmd: filebeat modules enable system`   Выполнить команду `filebeat modules enable system`
* `chdir: /usr/share/filebeat/bin`    Команду `filebeat modules enable system` выполнить в директории `/usr/share/filebeat/bin` 
* `register: filebeat_modules`    Результат выполнения записать в переменую `filebeat_modules`
* `changed_when: filebeat_modules.stdout != 'Module system is alredy enabled'`    Вывести сообщение `Module system is alredy enabled` в том случае, если модуль будет установлен в системе.

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
* `- name: Load Kibana dashboard`   Имя задачи.
* `become: true`       Выполнять с повышением привелегий (под УЗ root по дефолту)
* `command:`       Использование модуля `command`. Этот модуль выполняет системные команды на `manage_node`
* `cmd: filebeat setup`   Выполнить команду `filebeat setup`
* `chdir: /usr/share/filebeat/bin`    Команду `filebeat setup` выполнить в директории `/usr/share/filebeat/bin` 
* `register: filebeat_setup`    Результат выполнения записать в переменую `filebeat_setup`
* `changed_when: false`       Никогда не выводить сообщения об изменении состояния
* `until: filebeat_setup is succeeded`       Цикл для выполнения условий успешного выполнения запуска `filebeat setup`

9. Файл ` filebeat_roles/templates/filebeat.yml.j2 `
```yml
output.elasticsearch:
  hosts: ["http://{{ hostvars['el-instance']['ansible_facts']['default_ipv4']['address'] }}:9200"]
setup.kibana: 
  host: "http://{{ hostvars['k-instance']['ansible_facts']['default_ipv4']['address'] }}:5601"
filebeat.config.modules.path: ${path.config}/modules.d/*.yml
```
* `output.elasticsearch: hosts: ["http://{{ hostvars['el-instance']['ansible_facts']['default_ipv4']['address'] }}:9200"]` Переменная, определяет IP адрес хоста с Elasticsearch
* `setup.kibana: host: "http://{{ hostvars['k-instance']['ansible_facts']['default_ipv4']['address'] }}:5601"`  Переменная, определяет IP адрес хоста с Kibana
* `filebeat.config.modules.path: ${path.config}/modules.d/*.yml`        Путь к файлу конфигурации Kibana

10. Файл ` filebeat_roles/vars/main.yml `
```yml
filebeat_version: "7.14.0"
```
* переменная, задающая версию Filebeat
