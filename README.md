Vagrant-ansible-docker-prometheus-grafana
===================

Для начала работы необходимо подготовить систему. Установим все необходимые компоненты для работы.
Установим **Python3** и несколько библиотек:

    $ sudo apt update && sudo apt-get upgrade
    $ sudo apt install build-essential zlib1g-dev libncurses5-dev libgdbm-dev libnss3-dev libssl-dev libreadline-dev libffi-dev wget libfuse-dev
    $ sudo apt install python3
    $ curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
    $ python3 get-pip.py --user

Если все хорошо проверим результат, версии могут отличаться в зависимости от даты установки:

    $ python3 -m pip -V
    $ pip 23.1.2 from /home/ubuntudoc/.local/lib/python3.8/site-packages/pip (python 3.8)

Теперь очередь за **Ansible**, буду использовать версию 2.12.3:

    $ python3 -m pip install --user ansible-core==2.12.3
    $ python3 -m pip install --upgrade --user ansible
    $ sudo apt install ansible

Если все прошло удачно, то смотрим версию:

    $ ansible --version
    $ ansible [core 2.12.3]
    $ ...

Теперь для работы с виртуализацией, нужно установить **virtualbox** именно с помощью этого гипервизора будет осуществляться развертка виртуальных машин из Vagrant'a.
Для этого получим ключи для работы с virtualbox, добавим репозиторий и скачаем его, со всеми компонентами.

    $ wget -q https://www.virtualbox.org/download/oracle_vbox.asc -O- | sudo apt-key add -
    $ wget -q https://www.virtualbox.org/download/oracle_vbox_2016.asc -O- | sudo apt-key add -
    $ echo "deb [arch=amd64] http://download.virtualbox.org/virtualbox/debian $(lsb_release -cs) contrib" | sudo tee -a /etc/apt/sources.list.d/virtualbox.list
    $ sudo apt update && sudo apt-get install virtualbox virtualbox-ext-pack

Все прошло успешно, давайте посмотрим версию virtualbox

    $ echo $(virtualbox --help | head -n 1 | awk '{print $NF}')
    $ v6.1.38_Ubuntu

Отлично осталось установить **Vagrant** и можно приступать к работе.
Для устновки запаситесь терпением и VPN, для установки.
Скачиваем ключ, запишем репозиторий, обновим наши репозитории и установим сам Vagrant:

    $ wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
    $ echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
    $ sudo apt update && apt install vagrant

Еще для работы нам понадобиться SSH ключ для подключения к виртуальному узлу:
Для этого установим SSHD:

    $ sudo apt install openssh-server
    
Так же пропишем, чтобы он влючался вместе с ОС

    $ sudo systemctl enable sshd
    
Теперь выпустим наш ключ:

    $ ssh-keygen -t rsa
    
Ничего не пишем просто соглашаемся Enter
Готово теперь они лежат в папке:
``/home/%YOURUSER%/.ssh/id_rsa`` Наш приватный ключ. Его мы оставляем себе и никуда не деваем
и
``/home/%YOURUSER%/.ssh/id_rsa.pub`` А вот публичный ключ мы закинем в нашу виртаульную машину чтобы иметь доступ по ssh

**Готово!**

Vagrantfile
==========

Для начала работы с Vagrant давайте создадим папку куда уже будем помещать весь наш проект.

    $ mkdir test
    $ cd test
    
Сразу закинем наш публичный ключ:

    $ cp /home/%YOURUSER%/.ssh/id_rsa ./

Отлично! Теперь проинициализируем пустой файл где настроим нашу виртуальную машину:

    $ vagrant init
    
Теперь в этой директории появится файл Vagrantfile, давайте сразу зайдем в него и соберем нашу виртаульную машину:

    $ nano Vagrantfile
    
 ```bash
Vagrant.configure(2) do |config|			        # начинает цикл config
  config.vm.box = "generic/ubuntu2004"		        # указываем название образа из репозитория это Ubuntu 20.04, с официальным каталогом можно ознакомиться тут: https://app.vagrantup.com/generic/
  config.vm.box_check_update = false				# По-умолчания, vagrant перед каждым запуском образа проверяет репозиторий на наличие обновлений. 
  config.ssh.username="admin"                       # Указываем имя пользователя для подключения по ssh
  config.ssh.password="admin"                       # Указываем пароль для подключения по ssh
  config.vm.hostname = "testing"  				    # задаем hostname
  config.vm.network "public_network", ip: "192.168.0.197"	# задаем публичный адрес, с которым он выйдет к нам в сеть 
  config.vm.define "testing"					    # задаёт имя машины для vagrant и VBox
  
  config.vm.provider "virtualbox" do |vb|			# используем провайдер VBox без gui и что памяти мы готовы выделить не больше 1 Гб
     vb.gui = false
     vb.memory = "1024"
  config.vm.provision "shell", inline: "cat id_rsa.pub >> .ssh/authorized_keys"    # команда которая запишет наш публичный ssh ключ в файл узла
  config.vm.provision "ansible" do |ansible|			# Задаем какой плейбук отработать при запуске
    ansible.playbook = "ansible_vagrant_playbook.yml"    # это название нашего плейбука
  end
  end
end
  ```
``Нажимаем ctrl+o и enter чтобы сохранить и ctrl+x для выхода``

Наш Vagrantfile готов. Теперь остается дело за великим напишим наш плейбук, который на это машину установит нам
- Docker
- Docker-compose
- Node-exporter
- Закинет файл Docker-compose и запустит нужные нам контейнеры для работы:
      - Prometheus
      - Grafana

Ansible
==========

Для начала работы создадим ansible.cfg и настроим наш конфиг с ansible

    $ nano ansible.cfg
    
И пишем туда следующее

 ```bash
[defaults]                         # Означает что применять по-умолчанию
host_key_checking = False          # Отключает проверку ssh ключа
inventory=hosts.txt            # файл инвентарь откуда будут браться наши переменные доступа к хосту(для виртуальной машины)
interpreter_python=auto_silent    # Выбирает с помощью какого Python работать, можно переоапределить Если вы используете например Python2
localhost_warning=false            # По умолчанию Ansible выдает предупреждение, когда в инвентаре нет хостов. Эти предупреждения можно отключить, установив для этого параметра значение False
 ```

Теперь необходимо написать наш инвентори файл. Файл в котором будет прописано все к кому мы хотим обращаться:

    $ nano hosts.txt
    
Пишем туда следующее:
 ```bash
[pc]                                    # Название группы узлов
192.168.0.197                           # Ip адрес или ссылка на него например host.example.com

[pc:vars]                                 # Название группы и переменные для доступа. Тут записываются все переменные как достучаться до хоста
ansible_user=admin                        # Имя пользователя
ansible_become_password=admin             # Пароль
ansible_ssh_private_key_file=/home/%YOURUSERNAME%/.ssh/id_rsa    # И ключ по которому мы подключимся к узлу по ssh
 ```

Теперь перед началом написание самого плейбука напишем наши файлы для docker-compose и контейнеров:

docker-compose.yml
===========

Создадим наш docker-compose файл:

    $ nano docker-compose.yml
    
Туда пишем:
 ```bash
version: "3.9"        # Версия нашего файла, подробнее отличий версий можно прочитать тут: https://docs.docker.com/compose/compose-file/compose-versioning/
services:             # указываем что будет в нашем докер файле

  grafana:            # имя нашего образа "grafana"
    image: grafana/grafana:8.5.3-ubuntu      # Образ графаны версии 8.5.3 на убунту взят из репозитория docker: https://hub.docker.com/
    ports:
    - "3000:3000"                            # проводим порт контейнера(3000) до хоста на 3000 порту
    volumes:
      - grafana-data:/var/lib/grafana        # Подключаем наши диски, где хранить информацию о контейнере
      - grafana-configs:/etc/grafana
      - ./prometheus_ds.yml:/etc/grafana/provisioning/datasources/prometheus_ds.yml    # также прикрепляем наш файл с конфигурацией откуда собирать информацию(datasource) ниже мы его создадим
  prometheus:          # имя нашего образа "prometheus"
    image: prom/prometheus:v2.36.0           # Выбираем образ prometheus
    ports:
    - "9090:9090"                            # проводим порт контейнера(9090) до хоста на 9090 порту
    volumes:
    - prom-data:/prometheus                  # также подключим наш диск, для храния информации
    - ./prometheus.yml:/etc/prometheus/prometheus.yml    # также прикрепляем наш файл с конфигурацией откуда собирать информацию(там будет указан наш node-exporter) ниже мы его создадим

volumes:        # и создаем наши диски 
  grafana-data:
  grafana-configs:
  prom-data:
 ```

Теперь давайте же создадим файлы для нашего prometheus, и также для grafana

prometheus.yml.j2
===========

Создаем файл с названием prometheus.yml.j2, он будет шаблоном в нашем ansible-galaxy или ролью в ansible
Сам же файл будет указывать prometheus откуда собирать метрики

    $ nano prometheus.yml.j2

Пишем следующее
 ```bash
global:            # глобальные настройки как собирать информацию, сколько хранить и т.п.
  scrape_interval: 15s        # с каким интервалом собирать информацию

scrape_configs:        # здесь находится конфигурация работы prometheus
- job_name: node        # имя работы
  static_configs:        # указывает что неизменная конфигурация
  - targets: ['{{ ansible_host }}:9100']        # и цель откуда брать метрики
 ```
Немного пояснения
```bash
{{ ansible_host }} - это ссылка в ansible, то есть указывает что, подставить сюда из самого playbook, 
данная команда значит взять из hosts.txt указанный нами ip адрес узла. 
Все такие переменные указываются в двойных фигурных скобках "{{ }}"
 ```

prometheus_ds.yml.j2
===========

Создаем файл с названием prometheus_.yml.j2, он будет шаблоном в нашем ansible-galaxy или ролью в ansible

    $ nano prometheus_ds.yml.j2

Пишем следующее
 ```bash
datasources:            # Задаем grafana источник данных
- name: Prometheus      # имя Prometheus
  access: proxy         # доступ по прокси
  type: prometheus      # тип prometheus
  url: http://{{ ansible_host }}:9090    # и адрес откуда брать данные для визуализирования
  isDefault: true        # и укзываем по-умолчанию
 ```

Ansible-galaxy
==========

Теперь возвращаемся к Ansible. Для начала проинициализируем и назовем его ansible_vagrant:

    $ asnible init roles/ansible_vagrant

У нас появятся папки и файлы в них. Давайте посмотрим

    $ tree
    
 ```bash
test-
  ansible.cfg
  id_rsa.pub
  hosts.txt
  prometheus.yml.j2
  prometheus_ds.yml.j2
  docker-compose.yml
  Vagrantfile
      roles-
          ansible_vagrant-
                        defaults-
                            -main.yml
                        files-
                            none
                        handlers-
                            -main.yml
                        meta-
                            -main.yml
                        tasks-
                            -main.yml
                        templates-
                            none
                        tests-
                            -main.yml
                        vars-
                            -main.yml

Каждая папка отвечает за свои методы взаимодействия всего плейбука. 
подробности тут: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html            
```

Для начала мы сразу закинем наши файлы prometheus.yml.j2 и prometheus_ds.yml.j2 в папку roles/ansible_vagrant/templates/, а файл docker-compose.yml в папку roles/ansible_vagrant/files/

    $ mv prometheus.yml.j2 roles/ansible_vagrant/templates/
    $ mv prometheus_ds.yml.j2 roles/ansible_vagrant/templates/
    $ mv docker-compose.yml roles/ansible_vagrant/files/

Теперь приступи к написанию плейбука, который будет запускать наши роли создаем файл ansible_vagrant_playbook.yml

    $ nano ansible_vagrant_playbook.yml

И записываем туда:
```bash
- name: Install docker,docker-compose and node-exporter        # название которое несет весь плейбук
  hosts: all        # здесь указываются хосты, группы или группа хостов на которые будут производиться все возможны манипуляции
  become: yes        # Говорим ansible, чтобы он использовал авторизацию для получения прав работы над системой
  ignore_errors: yes        # так же игнорируем все ошибки, которые могут возникнуть в процессе плейбука, чтобы он не завершал работу, а дошел до конца

  roles:        # говорим ansible, что бы хотим запустить роль
    - ansible_vagrant        # и название роли
```

Сохраняем и идем изменять файл с переменными он находится в roles/ansible_vagrant/vars/main.yml
Эти переменные будут вызываться в теле нашего основного файла, где будут проходить все манипуляции

    $ nano roles/ansible_vagrant/vars/main.yml

```bash
---
ansible_user: admin        # для авторизации используем имя пользователя
ansible_become_password: admin    # и такой пароль
node_exporter_version: "1.1.2"    # используем версию node-exporter 1.1.2
node_exporter_bin: /usr/local/bin/node_exporter    # переменная директории node_exporter
node_exporter_user: node-exporter        # имя пользователя node_exporter
node_exporter_group: "{{ node_exporter_user }}" # название группы наследуем из node_exporter_user
node_exporter_dir_conf: /etc/node_exporter # местоположение конфигурационного файла node_exporter
...
```

Дальше мы добавим handlers, команды которые находится здесь вызываются, только тогда когда происходит изменение файла или команды или цикл команд.
Давайте изменим его и внесем наши команды, он содерится тут: roles\ansible_vagrant\handlers\main.yml:

    $ nano roles\ansible_vagrant\handlers\main.yml

```bash
---
- name: reload_daemon_and_restart_node_exporter        # название handler'a по которому мы будем его вызывать
  service: name=node_exporter state=restarted daemon_reload=yes enabled=yes    # здесь мы указываем. что нужно service с названием node_exporter перезагрузить

- name: restart_docker_compose            # название handler'a по которому мы будем его вызывать
  community.docker.docker_compose: project_src=/home restarted=true        # указываем, что будем перезагружать docker-compose файл в местоположении /home перезагружать
```

Давайте сразу напишем файл node_exporter.service, чтобы он включался и оставим его в местоположении roles\ansible_vagrant\templates\node_exporter.service.j2.

    $ nano roles\ansible_vagrant\templates\node_exporter.service.j2

```bash
[Unit]
Description=Node Exporter Version {{ node_exporter_version }}        # Добавляем описание службе
After=network-online.target        # Это гарантирует, что обычные сервисные единицы выполняют базовую инициализацию системы и завершаются корректно до завершения работы системы
[Service]            # указываем на службу
User={{ node_exporter_user }}        # от какого пользователя будет работать служба
Group={{ node_exporter_user }}        # и принадлежность группы пользователей
Type=simple            # тип установим в simple, это если диспетчер служб запустит службу после того, как основной процесс службы был остановлен
ExecStart={{ node_exporter_bin }}        # Наследуем местоположение из vars что запустить
[Install]        #    устанавливаем зависимость
WantedBy=multi-user.target        # на каком загрузочном таргете запускать этот юнит - многопользовательский режим
```

Сохраняем и остался последний, но не менее важный файл roles\ansible_vagrant\tasks\main.yml. Здесь хранится все тело нашего скрипта, что ему выполнять по нашему запросу.
Приступим:

    $ nano roles\ansible_vagrant\tasks\main.yml

```bash
---
- block: #====BLOCK INSTALL DOCKER AND DOCKER-COMPOSE FOR UBUNTU====    создаем блок, что мы будем здесь выполнять
    - name: Install aptitude using apt        # даем имя и с помощью apt устанавливаем пакет aptitude и также обновляем наш репозиторий
      apt:
        name=aptitude 
        state=latest 
        update_cache=yes 
        force_apt_get=yes

    - name: Install required system packages        # здесь мы указываем какие пакеты для работы нам необходимы, такие как python, https, curl и т.п.
      apt: name={{ item }} state=latest update_cache=yes
      loop: [ 'apt-transport-https', 'ca-certificates', 'curl', 'software-properties-common', 'python3-pip', 'virtualenv', 'python3-setuptools' ]

    - name: Add Docker GPG apt Key        # тут мы получаем gpg ключ для установки докера
      apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        state: present

    - name: Add Docker Repository        # добавляем репозиторий в систему и обновляем его
      apt_repository:
        repo: deb https://download.docker.com/linux/ubuntu focal stable
        state: present
        update_cache: yes

    - name: Update apt and install docker-ce    # наконец устанавливаем docker-ce, docker-ce-cli, containerd.io и docker-buildx-plugin
      apt: name={{ item }} state=latest
      loop: ['docker-ce','docker-ce-cli','containerd.io','docker-buildx-plugin']

    - name: Install docker-compose    # скачиваем docker-compose и помещаем его в /usr/local/bin/docker-compose назначая для него права выполнения
      get_url: 
        url : https://github.com/docker/compose/releases/download/1.25.1-rc1/docker-compose-Linux-x86_64
        dest: /usr/local/bin/docker-compose
        mode: 'u+x,g+x'

    - name: Install Docker Module for Python    # теперь устанавливаем docker и docker-compose
      pip:
        name={{ item }}
      loop: ['docker','docker-compose']

    - name: check docker is active        # проверяем все ли работает(проверяем сервис)
      service: name=docker state=started enabled=yes

- block: #====BLOCK INSTALL NODE-EXPORTER FOR UBUNTU====    тут идет блок инсталяции node-exporter
    - name: check if node exporter exist        # проверяем node_exporter на наличие в системе и выдаем результат
      stat:
        path: "{{ node_exporter_bin }}"
      register: __check_node_exporter_present

    - name: create node exporter user        # создаем пользователся node_exporter без фхоа и папки
      user:
        name: "{{ node_exporter_user }}"
        append: true
        shell: /usr/sbin/nologin
        system: true
        create_home: false

    - name: create node exporter config dir        # Создаем конфигурационную папку наследуя все параметры из vars
      file:
        path: "{{ node_exporter_dir_conf }}"
        state: directory
        owner: "{{ node_exporter_user }}"
        group: "{{ node_exporter_group }}"

    - name: if node exporter exist get version        # просматриваем версию node_exporter если он есть
      shell: "cat /etc/systemd/system/node_exporter.service | grep Version | sed s/'.*Version '//g"
      when: __check_node_exporter_present.stat.exists == true
      changed_when: false
      register: __get_node_exporter_version
      
    - name: download and unzip node exporter if not exist        # скачиваем сам node_exporter, распаковываем и помещаем его в /tmp/ на виртуальную машину
      unarchive:
        src: "https://github.com/prometheus/node_exporter/releases/download/v{{ node_exporter_version }}/node_exporter-{{ node_exporter_version }}.linux-amd64.tar.gz"
        dest: /tmp/
        remote_src: yes
        validate_certs: no

    - name: move the binary to the final destination    # теперь перемещаем этот в файл в /usr/local/bin/node_exporter если его не было в системе или не совпадают версии с правами 0755
      copy:
        src: "/tmp/node_exporter-{{ node_exporter_version }}.linux-amd64/node_exporter"
        dest: "{{ node_exporter_bin }}"
        owner: "{{ node_exporter_user }}"
        group: "{{ node_exporter_group }}"
        mode: 0755
        remote_src: yes
      when: __check_node_exporter_present.stat.exists == false or not __get_node_exporter_version.stdout == node_exporter_version

    - name: clean    # удаляем наш скачанный архив
      file:
        path: /tmp/node_exporter-{{ node_exporter_version }}.linux-amd64/
        state: absent

    - name: install service    # устанавливаем наш сервис, тут мы не указываем путь откуда взять т.к. ansible уже знает где взять template
      template:
        src: node_exporter.service.j2    # формат jinja2 позволяет внутри такого файла ссылаться на переменные playbook в отличии от отбычного файла, там такой возможности нет
        dest: /etc/systemd/system/node_exporter.service
        owner: root
        group: root
        mode: 0755
      notify: reload_daemon_and_restart_node_exporter    # при изменении файла наш node_exporter будет перезапущен
      
    - meta: flush_handlers    # это особый вид задач, которые могут влиять на внутреннее выполнение или состояние Ansible.
    
    - name: service always started            # И теперь просто запускаем наш node_exporter
      systemd: name=node_exporter state=started enabled=yes

- block: #====DOCKER-COMPOSE FILE PROMETHEUS/GRAFANA====    в этом блоке мы копируем наши файлы на виртуальную машину для выполнения docker-compose
    - name: copy compose file
      copy:
        src=docker-compose.yml
        dest=/home
        mode=0775
      notify: restart_docker_compose

    - name: prometheus file
      template:
        src: prometheus.yml.j2
        dest: /home/prometheus.yml
        owner: root
        group: root
        mode: 0755
      notify: restart_docker_compose

    - name: prometheus file datasource
      template:
        src: prometheus_ds.yml.j2
        dest: /home/prometheus_ds.yml
        owner: root
        group: root
        mode: 0755
      notify: restart_docker_compose

- block: #====DOCKER-COMPOSE UP====    и последнее мы просто собираем наш compose файл
    - name: deploy docker-compose stack
      community.docker.docker_compose:
        project_src: /home
        files: docker-compose.yml
        recreate: always
```

Finish!!!
==========
пишшем в консоле vagrant up и наблюдаем, как все собирается

    $ vagrant up

По итогу мы создали виртуальную машину на которой были установлены docker, docker-compose, node_exporter и был собран compose файл где настроены prometheus и grafana вместе, уже собирают метрики и визуализируют это все.
можно проверить на ip адресах:
192.168.0.197:9100 - node_exporter
192.168.0.197:9000 - prometheus
192.168.0.197:3000 - grafana
