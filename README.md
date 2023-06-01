# Prometheus - Grafana
Для выполнения этого действия требуется установить приложением git:
`git clone https://github.com/altyn-kenzhebaev/prometheus-hw14.git`
В текущей директории появится папка с именем репозитория. В данном случае prometheus-hw14. Ознакомимся с содержимым:
```
cd prometheus-hw14
ls -l
README.md
Vagrantfile
```
Здесь:
- README.md - файл с данным руководством
- Vagrantfile - файл описывающий виртуальную инфраструктуру для `Vagrant`
Запускаем ВМ:
```
vagrant up
```
Дальнейшая работы будет проведена под пользователем root:
```
sudo su -
```
## Установка Prometheus 
Устанавливаем вспомогательные пакеты и скачиваем Prometheus:
```
$ yum update -y
$ yum install wget vim -y
$ wget https://github.com/prometheus/prometheus/releases/download/v2.44.0/prometheus-2.44.0.linux-amd64.tar.gz
```
Создаем пользователя и нужные каталоги, настраиваем для них владельцев:
```
$ useradd --no-create-home --shell /bin/false prometheus
$ mkdir /etc/prometheus
$ mkdir /var/lib/prometheus
$ chown prometheus:prometheus /etc/prometheus
$ chown prometheus:prometheus /var/lib/prometheus
```
Распаковываем архив, для удобства переименовываем директорию и копируем бинарники в /usr/local/bin:
```
$ cd /
$ tar -xvzf /root/prometheus-2.44.0.linux-amd64.tar.gz
$ mv prometheus-2.44.0.linux-amd64 prometheuspackage
$ cp prometheuspackage/prometheus /usr/local/bin/
$ cp prometheuspackage/promtool /usr/local/bin/
```
Меняем владельцев у бинарников:
```
$ chown prometheus:prometheus /usr/local/bin/prometheus
$ chown prometheus:prometheus /usr/local/bin/promtool
```
По аналогии копируем библиотеки:
```
$ cp -r prometheuspackage/consoles /etc/prometheus
$ cp -r prometheuspackage/console_libraries /etc/prometheus
$ chown -R prometheus:prometheus /etc/prometheus/consoles
$ chown -R prometheus:prometheus /etc/prometheus/console_libraries
```
Создаем файл конфигурации:
```
$ vim /etc/prometheus/prometheus.yml
global:
  scrape_interval: 10s

scrape_configs:
  - job_name: 'prometheus_master'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
$ chown prometheus:prometheus /etc/prometheus/prometheus.yml
```
Настраиваем сервис:
```
$ vim /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target
[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
--config.file /etc/prometheus/prometheus.yml \
--storage.tsdb.path /var/lib/prometheus/ \
--web.console.templates=/etc/prometheus/consoles \
--web.console.libraries=/etc/prometheus/console_libraries
[Install]
WantedBy=multi-user.target!
$ systemctl daemon-reload
$ systemctl start prometheus
$ systemctl status prometheus
```
Убеждаемся, что prometheus успешно установлен и доступен по веб-интрефейсу http://192.168.50.10:9090/

![](Screenshot%20from%202023-06-01%2010-19-24.png)

## Установка Node Exporter
Скачиваем и распаковываем Node Exporter:
```
$ cd /root/
$ wget https://github.com/prometheus/node_exporter/releases/download/v1.5.0/node_exporter-1.5.0.linux-amd64.tar.gz
$ tar xzfv node_exporter-1.5.0.linux-amd64.tar.gz
```
Создаем пользователя, перемещаем бинарник в /usr/local/bin:
```
$ useradd -rs /bin/false nodeusr
$ mv node_exporter-1.5.0.linux-amd64/node_exporter /usr/local/bin/
```
Создаем сервис:
```
$ vim /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
After=network.target
[Service]
User=nodeusr
Group=nodeusr
Type=simple
ExecStart=/usr/local/bin/node_exporter
[Install]
WantedBy=multi-user.target
```
Запускаем сервис:
```
$ restorecon -rv /usr/local/bin/node_exporter
$ systemctl daemon-reload
$ systemctl start node_exporter
$ systemctl enable node_exporter
```
Обновляем конфигурацию Prometheus:
```
$ vim /etc/prometheus/prometheus.yml
global:
  scrape_interval: 10s

scrape_configs:
  - job_name: 'prometheus_master'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter_almalinux'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9100']
```
Перезапускаем сервис:
```
$ systemctl restart prometheus
```
Проверяем http://192.168.50.10:9090/targets

![](Screenshot%20from%202023-06-01%2010-43-57.png)

Проверяем http://192.168.50.10:9100/metrics

![](Screenshot%20from%202023-06-01%2010-45-28.png)

## Установка Grafana
Установка Grafana:
```
$ yum localinstall /root/grafana_enterprise_9.5.2_1.x86_64-364648-163733.rpm
```
Стартуем сервис:
```
$ systemctl daemon-reload
$ systemctl start grafana-server
```
Проверяем http://192.168.50.10:3000 (admin/admin), меняем пароль

![](Screenshot%20from%202023-06-01%2010-59-01.png)

Далее интеграция с Prometheus:
Home => Administration => Data Sources => Add data sources => Prometheus => URL: http://192.168.50.10:9090 => Save & test

Результат:

![](Screenshot%20from%202023-06-01%2011-04-15.png)

Создание Dashboard:
Home => New => Import => Import via grafana.com: 1860 => Load => prometheus: Prometheus => Import

Результат:

![](Screenshot%20from%202023-06-01%2011-09-17.png)

## Установка AlertManager
Скачиваем и распаковываем AlertManager
```
$ wget https://github.com/prometheus/alertmanager/releases/download/v0.25.0/alertmanager-0.25.0.linux-amd64.tar.gz
$ tar zxf alertmanager-0.25.0.linux-amd64.tar.gz
```
Создаем пользователя и нужные директории
```
$ useradd --no-create-home --shell /bin/false alertmanager
$ usermod --home /var/lib/alertmanager alertmanager
$ mkdir /etc/alertmanager
$ mkdir /var/lib/alertmanager
```
Копируем бинарники из архива в /usr/local/bin и меняем владельца
```
$ cp alertmanager-0.25.0.linux-amd64/amtool /usr/local/bin/
$ cp alertmanager-0.25.0.linux-amd64/alertmanager /usr/local/bin/
$ cp alertmanager-0.25.0.linux-amd64/alertmanager.yml /etc/alertmanager/
$ chown -R alertmanager:alertmanager /etc/alertmanager /var/lib/alertmanager
$ chown alertmanager:alertmanager /usr/local/bin/{alertmanager,amtool}
$ echo "ALERTMANAGER_OPTS=\"\"" > /etc/default/alertmanager
$ chown alertmanager:alertmanager /etc/default/alertmanager
$ chown -R alertmanager:alertmanager /var/lib/alertmanager
```
Настраиваем сервис
```
$ vim /etc/systemd/system/alertmanager.service
[Unit]
Description=Alertmanager Service
After=network.target prometheus.service

[Service]
EnvironmentFile=-/etc/default/alertmanager
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager \
          --config.file=/etc/alertmanager/alertmanager.yml \
          --storage.path=/var/lib/alertmanager \
          $ALERTMANAGER_OPTS
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
Restart=always

[Install]
WantedBy=multi-user.target
```
Запускаем сервис
```
$ systemctl daemon-reload
$ systemctl start alertmanager
```
Настраиваем правила
```
$ vim /etc/prometheus/rules.yml
groups:
- name: alert.rules
  rules:
  - alert: InstanceDown
    expr: up == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      description: '{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute.'
      summary: Instance {{ $labels.instance }} down
```
Обновляем конфиг prometheus:
```
$ vim /etc/prometheus/prometheus.yml
global:
  scrape_interval: 10s

scrape_configs:
  - job_name: 'prometheus_master'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter_almalinux'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9100']

rule_files:
  - "rules.yml"
alerting:
  alertmanagers:
    - static_configs:
      - targets:
        - localhost:9093
```
Рестартуем сервисы
```
$ systemctl restart prometheus
$ systemctl restart alertmanager
```
Тестируем проходим по ссылке http://192.168.50.10:9090/rules

![](Screenshot%20from%202023-06-01%2013-24-13.png)

Опускаем инстанс для теста alertmanager `systemctl stop prometheus`

![](Screenshot%20from%202023-06-01%2013-26-02.png)