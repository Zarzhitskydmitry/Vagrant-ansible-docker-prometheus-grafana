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
{{ ansible_host }} - это ссылка в ansible, то есть указывает что, подставить сюда из самого playbook, данная команда значит взять из hosts.txt указанный нами ip адрес узла. Все такие переменные указываются в двойных фигурных скобках "{{ }}"
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

Каждая папка отвечает за свои методы взаимодействия всего плейбука, подробности тут: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html            
```

Для начала мы сразу закинем наши файлы prometheus.yml.j2 и prometheus_ds.yml.j2 в папку roles/ansible_vagrant/templates/

    $ mv prometheus.yml.j2 roles/ansible_vagrant/templates/
    $ mv prometheus_ds.yml.j2 roles/ansible_vagrant/templates/

Ansible-galaxy
==========

Quick Start
===========

Our Wiki contains an `Introduction to the API <https://github.com/python-telegram-bot/python-telegram-bot/wiki/Introduction-to-the-API>`_ explaining how the pure Bot API can be accessed via ``python-telegram-bot``.
Moreover, the `Tutorial: Your first Bot <https://github.com/python-telegram-bot/python-telegram-bot/wiki/Extensions-%E2%80%93-Your-first-Bot>`_ gives an introduction on how chatbots can be easily programmed with the help of the ``telegram.ext`` module.

Resources
=========

- The `package documentation <https://docs.python-telegram-bot.org/>`_ is the technical reference for ``python-telegram-bot``.
  It contains descriptions of all available classes, modules, methods and arguments as well as the `changelog <https://docs.python-telegram-bot.org/changelog.html>`_.
- The `wiki <https://github.com/python-telegram-bot/python-telegram-bot/wiki/>`_ is home to number of more elaborate introductions of the different features of ``python-telegram-bot`` and other useful resources that go beyond the technical documentation.
- Our `examples section <https://docs.python-telegram-bot.org/examples.html>`_ contains several examples that showcase the different features of both the Bot API and ``python-telegram-bot``.
  Even if it is not your approach for learning, please take a look at ``echobot.py``. It is the de facto base for most of the bots out there.
  The code for these examples is released to the public domain, so you can start by grabbing the code and building on top of it.
- The `official Telegram Bot API documentation <https://core.telegram.org/bots/api>`_ is of course always worth a read.
