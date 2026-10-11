# Отчет по лабораторной работе 1
## Loki + Zabbix + Grafana

**Задача**

Подключить к тестовому сервису Nextcloud мониторинг + логирование. Осуществить визуализацию через Grafana

## 1. Логирование
1. Создаем файл `docker-compose.yml` аналогичный файлу из задания, чтобы объединить все необходимое для лабораторной.
   ```
   services:
      nextcloud:
        image: nextcloud:29.0.6
        container_name: nextcloud # на это имя будет завязана настройка забикса далее, так что лучше не менять
        ports:
          - "8080:80"
        volumes:
          - nc-data:/var/www/html/data
      loki: # сервис-обработчик логов
        image: grafana/loki:2.9.0
        container_name: loki
        ports:
          - "3100:3100"
        command: -config.file=/etc/loki/local-config.yaml # запуск с дефолтным конфигом
      promtail: # сервис-сборщик логов
        image: grafana/promtail:2.9.0
        container_name: promtail
        volumes:
          - nc-data:/opt/nc_data # та же самая директория, которая монтируется в Nextcloud
          - ./promtail_config.yml:/etc/promtail/config.yml
        command: -config.file=/etc/promtail/config.yml
      grafana: # сервис визуализации
        environment:
          - GF_PATHS_PROVISIONING=/etc/grafana/provisioning # просто
          - GF_AUTH_ANONYMOUS_ENABLED=true # дефолтные
          - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin # настройки
        command: /run.sh
        image: grafana/grafana:11.2.0
        container_name: grafana
        ports:
          - "3000:3000"
      postgres-zabbix:
        image: postgres:15
        container_name: postgres-zabbix
        environment:
          POSTGRES_USER: zabbix
          POSTGRES_PASSWORD: zabbix
          POSTGRES_DB: zabbix
        volumes:
          - zabbix-db:/var/lib/postgresql/data
        healthcheck:
          test: ["CMD", "pg_isready", "-U", "zabbix"]
          interval: 10s
          retries: 5
          start_period: 5s
      zabbix-server:
        image: zabbix/zabbix-server-pgsql:ubuntu-6.4-latest # непосредственно бэкенд забикса
        container_name: zabbix-back
        ports:
          - "10051:10051"
        depends_on:
          - postgres-zabbix
        environment:
          POSTGRES_USER: zabbix
          POSTGRES_PASSWORD: zabbix
          POSTGRES_DB: zabbix
          DB_SERVER_HOST: postgres-zabbix
      zabbix-web-nginx-pgsql:
        image: zabbix/zabbix-web-nginx-pgsql:ubuntu-6.4-latest # фронтенд забикса
        container_name: zabbix-front
        ports:
          - "8082:8080" # внешний порт можно любой по своему желанию
        depends_on:
          - postgres-zabbix
        environment:
          POSTGRES_USER: zabbix
          POSTGRES_PASSWORD: zabbix
          POSTGRES_DB: zabbix
          DB_SERVER_HOST: postgres-zabbix
          ZBX_SERVER_HOST: zabbix-back
    volumes:
      nc-data:
      zabbix-db:
   ```

   И конфигурационный файл promtail `promtail_config.yml`:
   ```
     server:
      http_listen_port: 9080
      grpc_listen_port: 0
    positions:
      filename: /tmp/positions.yaml
    clients:
      - url: http://loki:3100/loki/api/v1/push # адрес Loki, куда будут слаться логи
    scrape_configs:
      - job_name: system # любое имя
        static_configs:
          - targets:
              - localhost # т.к. монтируем папку с логами прямо в контейнер Loki, он собирает логи со своей локальной файловой системы
            labels:
              job: nextcloud_logs # любое имя, по этому полю будет осуществляться индексирование
              __path__: /opt/nc_data/*.log # необязательно указывать полный путь, главное сказать где искать log файлы
   ```
   
2. Запускаем наш docker-compose, проверяем, что все работает
   <img width="1109" height="250" alt="image" src="https://github.com/user-attachments/assets/d94cf29f-260b-4bd9-80fc-a0f1a8b4ffd9" />
   <img width="1125" height="261" alt="image" src="https://github.com/user-attachments/assets/877b7787-be83-49b9-bc82-3fb3d3059805" />

   <img width="1920" height="192" alt="image" src="https://github.com/user-attachments/assets/a663afa6-f4f6-4ce0-a00b-814e54dfddaf" />

Все запустилось

3. Инициализируем Nextcloud. Заходим на порт, который указан в compose файле и регистрируемся
<img width="1850" height="982" alt="image" src="https://github.com/user-attachments/assets/82958de2-2d2d-4f72-9026-57b16fe1d762" />

4. Проверяем в логах promtail, что он “подцепил” нужный нам log-файл: должны быть строчки, содержащие `msg=Seeked
/opt/nc_data/nextcloud.log`
<img width="1548" height="356" alt="image" src="https://github.com/user-attachments/assets/ea148f76-f822-49c9-89e4-408985407cae" />
Видим, что все так

## 2. Мониторинг
1. Заходим на порт Zabbix, который указали в compose, регистрируемся с кредами, указанными там же
<img width="1855" height="985" alt="image" src="https://github.com/user-attachments/assets/f7f5dbf5-af77-4a87-9b2c-a5c8b4b690ee" />

Вот так вот получается
<img width="1864" height="986" alt="image" src="https://github.com/user-attachments/assets/a9af9fc1-e8d6-4acc-a297-94100799e7f6" />

2. Создаем файл `template.yml` с кастомным шаблоном для мониторинга Nextcloud, аналогичный примеру из лабораторной
   ```
   zabbix_export:
    version: '6.4'
    template_groups:
      - uuid: a571c0d144b14fd4a87a9d9b2aa9fcd6
        name: Templates/Applications
    templates:
      - uuid: a615dc391a474a9fb24bee9f0ae57e9e
        template: 'Test ping template'
        name: 'Test ping template'
        groups:
          - name: Templates/Applications
        items:
          - uuid: a987740f59d54b57a9201f2bc2dae8dc
            name: 'Nextcloud: ping service'
            type: HTTP_AGENT
            key: nextcloud.ping
            value_type: TEXT
            trends: '0'
            preprocessing:
              - type: JSONPATH
                parameters:
                  - $.body.maintenance
              - type: STR_REPLACE
                parameters:
                  - 'false'
                  - healthy
              - type: STR_REPLACE
                parameters:
                  - 'true'
                  - unhealthy
            url: 'http://{HOST.HOST}/status.php'
            output_format: JSON
            triggers:
              - uuid: a904f3e66ca042a3a455bcf1c2fc5c8e
                expression: 'last(/Test ping template/nextcloud.ping)="unhealthy"'
                recovery_mode: RECOVERY_EXPRESSION
                recovery_expression: 'last(/Test ping template/nextcloud.ping)="healthy"'
                name: 'Nextcloud is in maintenance mode'
                priority: DISASTER
   ```

3. Добавляем этот шаблон в zabbix, после чего появляется плашка, что все успешно импортировалось
<img width="1868" height="201" alt="image" src="https://github.com/user-attachments/assets/48da1a5e-dbae-43d4-ac25-2334b8a9e069" />

4. Чтобы Zabbix и Nextcloud могли общаться по своим коротким именам внутри докеровской сети, в некстклауде необходимо “разрешить” это имя. Для этого
нужно зайти на контейнер некстклауда под юзером `www-data` и выполнить команду `php occ config:system:set trusted_domains 1 --value="nextcloud"`
<img width="879" height="95" alt="image" src="https://github.com/user-attachments/assets/57523f7c-8f5b-47f0-b1c0-3779b7ae29a7" />

Все работает

5. В Zabbix в разделе Data collection → Hosts делаем (Create host). Указываем адрес (имя) контейнера nextcloud, видимое имя - любое, хост группа - Applications (но в
целом можно любую другую). Чтобы не просто добавить хост, а начать его мониторинг, необходимо подключить к нему нужный шаблон мониторинга.
Поэтому в поле Templates нужно выбрать добавленный на шаге 2 Templates/Applications→ Test ping template

<img width="1305" height="720" alt="image" src="https://github.com/user-attachments/assets/d0d30793-21c0-49f7-9dfd-fc5b8e77854b" />

Теперь мы видим созданный нами хост в числе хостов

<img width="1859" height="705" alt="image" src="https://github.com/user-attachments/assets/ab2a44ea-ee37-4197-ab43-2250ddaf4b3a" />

6. Теперь переходим в раздел Monitoring → Latest data. Через какое-то время там должны появиться первые данные, в
нашем случае значение healthy

<img width="1861" height="819" alt="image" src="https://github.com/user-attachments/assets/6bd7e084-e845-44d1-a7de-ce9fb6087b44" />

Вот такая красота

7. Временно включаем в некстклауде maintenance mode ( `php occ maintenance:mode
--on` в контейнере), проверить что сработал триггер (раздел Monitoring → Problems), выключить режим обратно ( `php occ maintenance:mode --off` ), убедиться что
проблема помечена как “решенная”
  <img width="743" height="132" alt="image" src="https://github.com/user-attachments/assets/b6ee059d-07e8-4188-8594-71e65af45eb0" />

 <img width="1867" height="988" alt="image" src="https://github.com/user-attachments/assets/5e52b7a4-cf09-4cd5-89b9-4ccf9b901756" />

 <img width="1861" height="980" alt="image" src="https://github.com/user-attachments/assets/03b8b005-d052-414e-b1e5-10df813222bb" />

Видим, что все так

