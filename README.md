# Выполнение
## Подготовка


1. Два хоста:  
Elasticsearch+Kibana: Centos 7, 6Gb RAM, 20GB HDD  
Logstash: Centos 7, 4Gb RAM, 20GB HDD  
Я использовал yandex cloud.

2. Скачайте дистрибутив [java](https://www.oracle.com/java/technologies/javase-jdk11-downloads.html) и положите его в директорию `files/`. 

3. Внести соотвествующие адреса в inventory/prod.yml вместо *** (либо подготовить свою инфраструктуру, с хостами elasticsearch и logstash).

## Запуск
> ansible-playbook -i inventory/prod.yml site.yml

## Проверка
С хоста, на котором установлен Logstash, либо любого другого в этой же сети (в этом случае localhots нужно заменить на внешний адрес хоста Logstash)

> echo 'new message' | nc localhots 5432

Проверяем создание индекса. На хосте с Elasticsearch^
> curl -XGET 'http://localhost:9200/my-logs/_search?pretty'

## P.S.
1. Kibana не включал как сервис, можно запустить вручную с хоста elasticsearch под пользователем, с которого разворачивался ansible. Создание сервиса продемонстрировал в кейсах с elasticsearch и logstash.

3. Сервисы стартуют от имени пользователя, с которого разворачивался ansible.

2. На продуктиве лучше создать отдельных пользователей для запуска сервисов. Сейчас не стал это делать в целях экономии времени (и не усложнять site.yml)


# Комментарии

## Каталоги  

```

├── files
│   └── jdk-11.0.13_linux-x64_bin.tar.gz    --дистрибутив java
├── group_vars
│   ├── all                                 
│   │   └── vars.yml                        --vars для всех хостов                    
│   ├── elasticsearch
│   │   └── vars.yml                        --vars для хоста elasticsearch (jdk+elastic+kibana)
│   └── logstash
│       └── vars.yml                        --vars для хоста logstash (jdk+logstash)
├── inventory
│   └── prod.yml
├── site.yml
└── templates
    ├── elasticsearch
    │   ├── elasticsearch.service.j2        --темплейт сервиса для elasticsearch 
    │   ├── elasticsearch.yml.j2            --конфиг elasticsearch
    │   └── elk.sh.j2                       --переменные окружения для elasticsearch
    ├── jdk
    │   └── jdk.sh.j2                       --переменные окружения для jdk
    ├── kibana
    │   └── kib.sh.j2                       --переменные окружения для kibana
    └── logstash
        ├── lgs.sh.j2
        ├── logstash.service.j2             --темплейт сервиса для logstash
        ├── my_pipeline.conf.j2             --темплейт пайплайна logstash. Слушает порт 5432 и передает на хост с elasticsearch (определен как переменная в group_vars/logstash/vars.yml с именем elasticsearch_ip)
        └── pipelines.yml.j2                --темплейт pipelines.yml
```
## site.yml
```yml
---
- name: Install Java
  hosts: all
  tasks:
    - name: Set facts for Java 11 vars
      set_fact:
        java_home: "/opt/jdk/{{ java_jdk_version }}"
      tags: java
    - name: Upload .tar.gz file containing binaries from local storage
      copy:
        src: "{{ java_oracle_jdk_package }}"
        dest: "/tmp/jdk-{{ java_jdk_version }}.tar.gz"
      register: download_java_binaries
      until: download_java_binaries is succeeded
      tags: java
    - name: Ensure installation dir exists
      become: true
      file:
        state: directory
        path: "{{ java_home }}"
      tags: java
    - name: Extract java in the installation directory
      become: true
      unarchive:
        copy: false
        src: "/tmp/jdk-{{ java_jdk_version }}.tar.gz"
        dest: "{{ java_home }}"
        extra_opts: [--strip-components=1]
        creates: "{{ java_home }}/bin/java"
      tags:
        - java
    - name: Export environment variables
      become: true
      template:
        src: templates/jdk/jdk.sh.j2
        dest: /etc/profile.d/jdk.sh
      tags: java
- name: Install Elasticsearch
  hosts: elasticsearch
  handlers:
    - name: restart Elasticsearch                       <-- перезагрузка сервиса elasticsearch
      become: true
      service:
        name: elasticsearch
        state: restarted
  tasks:
    - name: Upload tar.gz Elasticsearch from remote URL
      get_url:
        url: "https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-{{ elastic_version }}-linux-x86_64.tar.gz"
        dest: "/tmp/elasticsearch-{{ elastic_version }}-linux-x86_64.tar.gz"
        mode: 0755
        timeout: 60
        force: false
        validate_certs: false
      register: get_elastic
      until: get_elastic is succeeded
      tags: elastic
    - name: Create directrory for Elasticsearch
      become: true
      file:
        state: directory
        path: "{{ elastic_home }}"
        owner: "{{ ansible_user_id }}"                      <-- доступ к директории elasticsearch для текущего пользователя
      tags: elastic
    - name: Extract Elasticsearch in the installation directory
      unarchive:
        copy: false
        src: "/tmp/elasticsearch-{{ elastic_version }}-linux-x86_64.tar.gz"
        dest: "{{ elastic_home }}"
        extra_opts: [--strip-components=1]
        creates: "{{ elastic_home }}/bin/elasticsearch"
      tags:
        - elastic
    - name: Set environment Elastic
      become: true
      template:
        src: templates/elasticsearch/elk.sh.j2
        dest: /etc/profile.d/elk.sh
      tags: elastic
    - name: Configure Elastic
      become: true
      template:
        src: templates/elasticsearch/elasticsearch.yml.j2
        dest: "{{ elastic_home }}/config/elasticsearch.yml"
      tags: elastic
    - name: Create elasticsearch service                    <-- создаем сервис elasticsearch
      become: true
      template:
        src: templates/elasticsearch/elasticsearch.service.j2
        dest: "/etc/systemd/system/elasticsearch.service"
      tags: elastic
      notify: restart Elasticsearch
    - name: enable Elasticsearch service                    <-- включаем сервис elasticsearch
      become: true
      service:
        name: elasticsearch
        enabled: yes
        masked: no
      notify: restart Elasticsearch                         <-- перезагружаем сервис elasticsearch
- name: Install Kibana
  hosts: elasticsearch
  tasks:
    - name: Upload tar.gz Kibana from remote URL
      get_url:
        url: "https://artifacts.elastic.co/downloads/kibana/kibana-{{ kibana_version }}-linux-x86_64.tar.gz"
        dest: "/tmp/kibana-{{ kibana_version }}-linux-x86_64.tar.gz"
        mode: 0755
        timeout: 60
        force: false
        validate_certs: false
      register: get_kibana
      until: get_kibana is succeeded
      tags: kibana
    - name: Create directrory for Kibana
      become: true
      file:
        state: directory
        path: "{{ kibana_home }}"
        owner: "{{ ansible_user_id }}"
      tags: kibana
    - name: Extract Kibana in the installation directory
      unarchive:
        copy: false
        src: "/tmp/kibana-{{ kibana_version }}-linux-x86_64.tar.gz"
        dest: "{{ kibana_home }}"
        extra_opts: [--strip-components=1]
        creates: "{{ kibana_home }}/bin/kibana"
      tags:
        - kibana
    - name: Set environment Kibana
      become: true
      template:
        src: templates/kibana/kib.sh.j2
        dest: /etc/profile.d/kib.sh
      tags: kibana
- name: Install Logstash
  hosts: logstash
  handlers:
    - name: restart Logstash
      become: true
      service:
        name: logstash
        state: restarted
  tasks:
    - name: Upload tar.gz Logstash from remote URL
      get_url:
        url: "https://artifacts.elastic.co/downloads/logstash/logstash-{{ logstash_version }}-linux-x86_64.tar.gz"
        dest: "/tmp/logstash-{{ logstash_version }}-linux-x86_64.tar.gz"
        mode: 0755
        timeout: 60
        force: false
        validate_certs: false
      register: get_logstash
      until: get_logstash is succeeded
      tags: logstash
    - name: Create directrory for Logstash
      become: true
      file:
        state: directory
        path: "{{ logstash_home }}"
        owner: "{{ ansible_user_id }}"
      tags: logstash
    - name: Extract Logstash in the installation directory
      unarchive:
        copy: false
        src: "/tmp/logstash-{{ logstash_version }}-linux-x86_64.tar.gz"
        dest: "{{ logstash_home }}"
        extra_opts: [--strip-components=1]
        creates: "{{ logstash_home }}/bin/logstash"
      tags:
        - logstash
    - name: Set environment Logstash
      become: true
      template:
        src: templates/logstash/lgs.sh.j2
        dest: /etc/profile.d/lgs.sh
      tags: logstash
    - name: Add pipelines.yml
      become: true
      template:
        src: templates/logstash/pipelines.yml.j2
        dest: "{{ logstash_home }}/config/pipelines.yml"
      tags: logstash
    - name: Add my_pipeline.conf
      become: true
      template:
        src: templates/logstash/my_pipeline.conf.j2
        dest: "{{ logstash_home }}/config/my_pipeline.conf"
      tags: logstash
    - name: Create logstash service
      become: true
      template:
        src: templates/logstash/logstash.service.j2
        dest: "/etc/systemd/system/logstash.service"
      tags: elastic
      notify: restart Logstash
    - name: enable Logstash service
      become: true
      service:
        name: logstash
        enabled: yes
        masked: no
      notify: restart Logstash

```
