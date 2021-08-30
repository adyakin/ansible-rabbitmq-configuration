ansible-rabbitmq-configuration
=========

Роль для настройки пользователей/хостов/политик на существующих кластерах. Используется совместно с ansible-rabbitmq


Host Variables
--------------

| variable | required | default | description |
| :------  | :-----:  | :-----: | :---------- |
| `rabbitmq_cluster_name` | - | - | имя кластера |
| `rabbitmq_external_endpoint` | - | - | внешний адрес кластера (например за haproxy). Используется при настройке федерации.

Непосредственно конфигурация для каждого vhost загружается динамически из `vars/vhost/*.yml`

Формат файла конфигурации:

```yaml
vhost-name:
  name: vhost-name
  state: present
  tracing: no
  rabbitmq_users:
    - user: admin
      password: admin
      tags:
        - administrator
    - user: fadmin
      password: fadmin
      tags:
        - administrator
    - user: test
      password: test

  rabbitmq_policies:
    - name: "ha-policy"
      pattern: ".*" # Optional, defaults to ".*"
      tags: # Optional, defaults to "{}"
        ha-mode: all
        ha-sync-mode: automatic
      state: present

```




Example Playbook
----------------

Если в inventory указаны только мастера на каждом кластере:

```yaml
- name: configure clusters
  hosts: masters
  become: yes

  roles:
    - ansible-rabbitmq-configuration
```

Для использования с тем же инвентарем, что и для `ansible-rabbitmq` необходимо добавить создание новой группы - по одному хосту из каждого кластера.

```yaml
- name: gather facts
  hosts: all
  tasks:

  - setup:

  - name: get cluster name
    command: "rabbitmqctl cluster_status | grep -q 'Cluster Name' | awk '{print \"$NF\"}'"
    register: status_cmd

  - name: set cluster name variable
    set_fact:
      cluster_name: "{{ status_cmd.stdout }}"
    when: not status_cmd.failed

  - name: show cluster name
    debug:
      msg: "cluster_name = {{ cluster_name }}"

  - name: fail if discovered cluster name is different from inventory defined
    fail:
      msg: >
        "Inconsistent cluster state: inventory defined name 
        '{{ rabbitmq_cluster_name }}', discovered - '{{ cluster_name}}'"
    when: >
      cluster_name != rabbitmq_cluster_name

# собираем новую группу хостов в которой будет по одному хосту из каждого кластера
- name: prepare host group
  connection: local
  host: localhost
  become: no
  gather_facts: no

  tasks:

  # проходим по всех хостам и добавляем в новый dict {cluster_name: hostname}
  # в случае дублирования ключа значение заменяется, в результате чего получаем
  # словарь с уникальными хостами для каждого кластера
  - name: set temporary host group
    set_fact:
      tmp_hosts: "{{ tmp_hosts|default({}) | combine( {hostvars[item].cluster_name : item} ) }}"
    with_items: "{{ groups.all }}"

  - name: add hosts to group
    add_host:
      group: tmp_masters
      hostname: "{{ item.value }}"
    loop: "{{ tmp_hosts|dict2items }}"

  - name: show tmp_masters group hosts
    debug:
      msg: "host = {{ item }}"
    with_inventory_hostnames:
      - tmp_masters

# настраиваем конфигурацию юзеров/хостов/политики на каждом кластере
- name: configure clusters
  hosts: tmp_masters
  become: yes

  roles:
    - ansible-rabbitmq-configuration

```

License
-------

BSD

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
