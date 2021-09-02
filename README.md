ansible-rabbitmq-configuration
=========

Роль для настройки пользователей/хостов/политик на существующих кластерах. Используется совместно с ansible-rabbitmq


Host Variables
--------------

| variable | required | default | description |
| :------  | :-----:  | :-----: | :---------- |
| `rabbitmq_cluster_name` | - | - | имя кластера |
| `rabbitmq_external_endpoint` | - | - | внешний адрес кластера (например за haproxy). Используется при настройке федерации.
| `rabbitmq_federation` | X | `no` | настраивать ли федерацию между хостами. Если влючено, то обязательно указание `rabbitmq_external_endpoint` |

Непосредственно конфигурация для каждого vhost загружается динамически из `vars/vhost/*.yml`

Формат файла конфигурации:

> Корневым елементом обязательно должно быть уникальное имя, в противном случае
> переменные затрут уже существующую конфигурацию

> Для настройки федерации и создания очередей используется первый определенный 
> пользователь

| variable | required     | default   | description |
| :------  | :-----:      | :-----:   | :---------- |
| `name`    |    X        |    -      | имя vhost   |
| `state`    |    -       | `present` | состояние   |
| `tracing`  |    -       |   `no`    | вкл/выкл трассировку |
| `federation` |  -       |   `no`    | для включения федерации между кластерами |
| `federation_opts` | -   |  -        | параметры апстрима https://www.rabbitmq.com/federation-reference.html#upstreams. В именах полей `"-"` необходимо заменить на  `"_"` |
| `rabbitmq_users` |  X   |   -       | список пользователей, возможные параметры - аналогично [community.rabbitmq.rabbitmq_user](https://docs.ansible.com/ansible/latest/collections/community/rabbitmq/rabbitmq_user_module.html#parameters) за исключением блока `permissions` |
| `rabbitmq_policies` | X  |  -       | политики, применяемые на уровне vhost. параметры: [community.rabbitmq.rabbitmq_policy](https://docs.ansible.com/ansible/latest/collections/community/rabbitmq/rabbitmq_policy_module.html#parameters) |
| `rabbitmq_exchanges`| X |  -        | параметры очередй, подробно в comminuty.rabbitmq.rabbitmq_exchange |
| `rabbitmq_queues`| X |  -        | параметры очередй, подробно в comminuty.rabbitmq.rabbitmq_queue |
| `rabbitmq_bindings`| X |  -        | параметры очередй, подробно в comminuty.rabbitmq.rabbitmq_binding |

Пример описания vhost `vhost-name`:

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

  rabbitmq_exchanges:
    - name: test-exchange
      type: direct
      state: present

  rabbitmq_queues:
    - name: queue1

  rabbitmq_bindings:
    - name: test-exchange
      destination: queue1
      destination_type: queue
      routing_key: "#"

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
---

- name: gather facts
  hosts: all
  tasks:

  - name: get cluster name
    command: "rabbitmqctl cluster_status --formatter json"
    changed_when: false
    register: status_cmd

  - name: set cluster name variable
    set_fact:
      cluster_name: "{{ status_cmd.stdout | from_json | json_query('cluster_name') }}"
    when: not status_cmd.failed

  - name: show cluster name
    debug:
      msg: "host = {{ ansible_inventory_hostname }}, cluster_name = {{ cluster_name }}"


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
