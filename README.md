# Настройка мониторинга  
Пошаговая инструкция выполнения домашнего задания.  
Настроить дашборд с 4-мя графиками:
- память
- процессор
- диск
- сеть  
  
Настройка на системе prometheus - grafana.  
В качестве результата прислать скриншот экрана - дашборд должен содержать в названии имя приславшего.

## Подготовка сервера  
Настроим некоторые параметры сервера, необходимые для правильно работы системы.
#### Время  
``yum install chrony``  
``systemctl enable chronyd``  
``systemctl start chronyd``  

#### При использовании фаервола необходимо открыть порты:  

TCP 9090 — http для сервера прометеус  
TCP 9093 — http для алерт менеджера  
TCP и UDP 9094 — для алерт менеджера  
TCP 9100 — для node_exporter  

``firewall-cmd --permanent --add-port=9090/tcp --add-port=9093/tcp --add-port=9094/{tcp,udp} --add-port=9100/tcp``  
``firewall-cmd --reload``

#### Отключим Selinux  
``setenforce 0``  

### Установка Prometheus  
С официальной страницы загрузки копируем подходящую нам ссылку и скачиваем:  
``wget https://github.com/prometheus/prometheus/releases/download/v2.37.0/prometheus-2.37.0.linux-amd64.tar.gz``

Создадим каталоги, по которым распределим файлы Prometheus  
``mkdir /etc/prometheus``  
``mkdir /var/lib/prometheus``  

Распакуем архив  
``tar xvf prometheus-2.37.0.linux-amd64.tar.gz``  

Перейдем в каталог с распакованными файлами  
``cd prometheus-*.linux-amd64``  

Распределим файлы по каталогам  
``cp prometheus promtool /usr/local/bin/``  
``cp -r console_libraries consoles prometheus.yml /etc/prometheus``  

#### Назначение прав
Создаем пользователя, от которого будем запускать систему мониторинга  
``useradd --no-create-home --shell /bin/false prometheus``  
*- мы создали пользователя "prometheus" без домашней директории и без возможности входа в консоль сервера.*  

Задаем владельца для каталогов, которые мы создали на предыдущем шаге  
``chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus``  

Задаем владельца для скопированных файлов  
``chown prometheus:prometheus /usr/local/bin/{prometheus,promtool}``

#### Запуск и проверка  

Запускаем prometheus командой:

``/usr/local/bin/prometheus --config.file /etc/prometheus/prometheus.yml --storage.tsdb.path /var/lib/prometheus/ --web.console.templates=/etc/prometheus/consoles --web.console.libraries=/etc/prometheus/console_libraries``  

Открываем веб-браузер и переходим по адресу  
``http://<IP-адрес сервера>:9090``
*Должна загрузиться консоль Prometheus*  

#### Автозапуск  
Для настройки автоматического старта Prometheus создадим новый юнит в systemd  
``vi /etc/systemd/system/prometheus.service``

[Unit]
Description=Prometheus Service
After=network.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
--config.file /etc/prometheus/prometheus.yml \
--storage.tsdb.path /var/lib/prometheus/ \
--web.console.templates=/etc/prometheus/consoles \
--web.console.libraries=/etc/prometheus/console_libraries
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target  

Перечитываем конфигурацию systemd  
``systemctl daemon-reload  
Разрешаем автозапуск  
``systemctl enable prometheus``  

После ручного запуска мониторинга, который мы делали для проверки, могли сбиться права на папку библиотек — снова зададим ей владельца  
``chown -R prometheus:prometheus /var/lib/prometheus``  

Запускаем службу:  
``systemctl start prometheus``  

... и проверяем, что она запустилась корректно  
``systemctl status prometheus`` 


## Установка Node_Exporter

Скачать архив  
``wget https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz``  
Распаковать  
``tar -xvf node_exporter-1.3.1.linux-amd64.tar.gz``  
Перейдем в каталог с распакованными файлами  
``cd node_exporter-*.linux-amd64``  
Копируем исполняемый файл в bin  
``cp node_exporter /usr/local/bin/``  
Создаем пользователя nodeusr  
``useradd --no-create-home --shell /bin/false nodeusr``  
Задаем владельца для исполняемого файла  
``chown -R nodeusr:nodeusr /usr/local/bin/node_exporter``  
### Автозапуск
Создаем файл node_exporter.service в systemd  
``vi /etc/systemd/system/node_exporter.service``  

[Unit]
Description=Node Exporter Service
After=network.target

[Service]
User=nodeusr
Group=nodeusr
Type=simple
ExecStart=/usr/local/bin/node_exporter
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target  

Перечитываем конфигурацию systemd  
``systemctl daemon-reload``  
Разрешаем автозапуск  
``systemctl enable node_exporter``  
Запускаем службу  
``systemctl start node_exporter``
Открываем веб-браузер и переходим по адресу  
``http://<IP-адрес сервера или клиента>:9100/metrics``  
— мы увидим метрики, собранные node_exporter


## Установка Grafana  

Создаем файл конфигурации репозитория для графаны  
``vi /etc/yum.repos.d/grafana.repo``

[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt  

Теперь можно устанавливать  
``yum install grafana``  
- отвечая "y" на все запросы  

### Настройка брандмауэра  
По умолчанию, Grafana работает на порту 3000. Для возможности подключиться к серверу открываем данный порт в фаерволе:  

``firewall-cmd --permanent --add-port=3000/tcp``  
``firewall-cmd --reload``  

### Запуск сервиса  
Разрешаем автозапуск  
``systemctl enable grafana-server``  
Запускаем  
``systemctl start grafana-server``  

Проверяем работу портала, открываем браузер и переходим по адресу  
``http://<IP-адрес сервера>:3000``  

Для авторизации используем логин и пароль: admin / admin.  
Система может потребовать задать новый пароль — вводим его дважды.  

Добавление источника данных  
Кликаем по иконке Configuration - Data Sources:  
Среди списка источников данных находим и выбираем Prometheus, кликнув по Select  

Задаем параметры для подключения к Prometheus:  
name: Prometheus  
url: http://localhost:9090  
Access: Server (default)  
* в данном примере мы оставили имя Prometheus и указали в качестве адреса локальный сервер (localhost). В случае, если графана и прометеус находятся на разных серверах, необходимо указать IP-адреса сервера Prometheus.  
Сохраняем настройки, кликнув по Save & Test  
Если мы все сделали правильно, система покажет сообщение «Data source is working»  

### Создание dashboard   
Переходим по Dashboard/New Dashboard/Add New Panel для создания новой панели
В качестве источника данных выбираем созданный ранее Prometheus
выбираем конкретные параметры и тип графика. После сохраняем настройку
