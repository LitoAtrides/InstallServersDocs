# Полная инструкция по установке Wazuh на Debian 13 с Docker (Elasticsearch 8.x)

## Предварительные требования

- Debian 13
- 4+ ГБ оперативной памяти
- 50+ ГБ свободного места на диске
- Интернет для загрузки пакетов и Docker образов

---

## ЭТАП 1: Подготовка системы

```bash
# Обновляем систему
sudo apt update
sudo apt upgrade -y

# Устанавливаем необходимые зависимости
sudo apt install -y curl gnupg2 lsb-release apt-transport-https ca-certificates ufw
```

---

## ЭТАП 2: Установка Docker и Docker Compose

```bash
# Устанавливаем Docker
sudo apt install -y docker.io docker-compose

# Добавляем текущего пользователя в группу docker
sudo usermod -aG docker $USER

# Активируем группу
newgrp docker

# Проверяем версию
docker --version
docker-compose --version

# Запускаем Docker демон
sudo systemctl enable docker
sudo systemctl start docker
```

---

## ЭТАП 3: Установка Wazuh Manager

```bash
# Добавляем GPG ключ Wazuh
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import -

# Исправляем права
sudo chmod 644 /usr/share/keyrings/wazuh.gpg

# Добавляем репозиторий
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list

# Обновляем список пакетов
sudo apt update

# Устанавливаем Wazuh Manager
sudo apt install -y wazuh-manager

# Запускаем сервис
sudo systemctl daemon-reload
sudo systemctl enable wazuh-manager
sudo systemctl start wazuh-manager

# Проверяем статус
sudo systemctl status wazuh-manager
```

---

## ЭТАП 4: Установка Elasticsearch 8.x

```bash
# Добавляем GPG ключ Elasticsearch
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

# Добавляем репозиторий для 8.x
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list

# Обновляем список пакетов
sudo apt update

# Устанавливаем последнюю версию Elasticsearch 8.x
sudo apt install -y elasticsearch
```

### Подготовка директорий Elasticsearch

```bash
# Создаем директорию для PID файла
sudo mkdir -p /var/run/elasticsearch
sudo chown elasticsearch:elasticsearch /var/run/elasticsearch
sudo chmod 755 /var/run/elasticsearch

# Очищаем старые данные и логи
sudo rm -rf /var/lib/elasticsearch/*
sudo rm -rf /var/log/elasticsearch/*
sudo rm -rf /var/lib/elasticsearch/.elasticsearch

# Даем правильные права
sudo chown -R elasticsearch:elasticsearch /var/lib/elasticsearch
sudo chown -R elasticsearch:elasticsearch /var/log/elasticsearch
sudo chown -R elasticsearch:elasticsearch /etc/elasticsearch

sudo chmod 750 /var/lib/elasticsearch
sudo chmod 750 /var/log/elasticsearch
sudo chmod 750 /etc/elasticsearch

sudo chmod 640 /etc/elasticsearch/elasticsearch.yml
sudo chmod 640 /etc/elasticsearch/jvm.options
```

### Конфигурация Elasticsearch

```bash
# Создаем конфиг с отключенной безопасностью (для разработки)
sudo bash -c 'cat > /etc/elasticsearch/elasticsearch.yml << "EOF"
cluster.name: wazuh
node.name: node-1
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: 0.0.0.0
http.port: 9200
discovery.type: single-node
xpack.security.enabled: false
xpack.watcher.enabled: false
xpack.ml.enabled: false
xpack.security.transport.ssl.enabled: false
xpack.security.http.ssl.enabled: false
EOF'

# Проверяем конфиг
sudo cat /etc/elasticsearch/elasticsearch.yml
```

### Настройка памяти JVM

```bash
# Используем sed для изменения параметров памяти
sudo sed -i 's/^-Xms1g/-Xms512m/' /etc/elasticsearch/jvm.options
sudo sed -i 's/^-Xmx1g/-Xmx512m/' /etc/elasticsearch/jvm.options

# Проверяем изменения
grep "^-Xm" /etc/elasticsearch/jvm.options
```

### Запуск Elasticsearch

```bash
# Перезагружаем systemd
sudo systemctl daemon-reload

# Включаем автозапуск
sudo systemctl enable elasticsearch

# Запускаем
sudo systemctl start elasticsearch

# Ждем 30 секунд на инициализацию
sleep 30

# Проверяем статус
sudo systemctl status elasticsearch

# Тестируем подключение (должен вернуть JSON)
curl http://localhost:9200
```

**Ожидаемый результат:**
```json
{
  "name" : "node-1",
  "cluster_name" : "wazuh",
  "cluster_uuid" : "...",
  "version" : {
    "number" : "8.19.6"
  },
  "tagline" : "You Know, for Search"
}
```

---

## ЭТАП 5: Установка Kibana из Docker

```bash
# Создаем директорию для проекта
mkdir -p ~/wazuh-docker
cd ~/wazuh-docker

# Загружаем образ Docker
docker pull docker.elastic.co/kibana/kibana:8.19.6

# Запускаем контейнер
docker run -d \
  --name kibana \
  --network host \
  -e ELASTICSEARCH_HOSTS=http://localhost:9200 \
  -e xpack.security.enabled=false \
  -e xpack.alerting.enabled=false \
  -e xpack.watcher.enabled=false \
  docker.elastic.co/kibana/kibana:8.19.6

# Ждем 40 секунд на инициализацию
sleep 40

# Проверяем статус контейнера
docker ps | grep kibana

# Смотрим логи
docker logs kibana

# Тестируем (должен вернуть HTML)
curl http://localhost:5601 | head -20
```

**Доступ:** http://ВАШЕ_IP:5601

### Альтернатива: Использовать docker-compose

```bash
# Создаем docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  kibana:
    image: docker.elastic.co/kibana/kibana:8.19.6
    container_name: kibana
    network_mode: "host"
    environment:
      - ELASTICSEARCH_HOSTS=http://localhost:9200
      - xpack.security.enabled=false
      - xpack.alerting.enabled=false
      - xpack.watcher.enabled=false
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5601"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 40s
EOF

# Запускаем
docker-compose up -d

# Проверяем логи
docker-compose logs -f kibana
```

---

## ЭТАП 6: Установка Wazuh Dashboard из Docker

```bash
# Добавляем GPG ключ Wazuh для Dashboard
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import -

# Исправляем права
sudo chmod 644 /usr/share/keyrings/wazuh.gpg

# Добавляем репозиторий Wazuh Dashboard
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh-dashboard.list

# Обновляем список пакетов
sudo apt update

# Загружаем образ Docker для Wazuh Dashboard (официальный образ)
docker pull wazuh/wazuh-dashboard:latest

# Запускаем Dashboard на порту 5602
docker run -d \
  --name wazuh-dashboard \
  --network host \
  -e INDEXER_URL=http://localhost:9200 \
  -e INDEXER_USERNAME=admin \
  -e INDEXER_PASSWORD=admin \
  -e WAZUH_API_URL=http://localhost:55000 \
  -e WAZUH_API_USERNAME=wazuh \
  -e WAZUH_API_PASSWORD=wazuh \
  wazuh/wazuh-dashboard:latest

# Ждем 40 секунд на инициализацию
sleep 40

# Проверяем статус
docker ps | grep wazuh-dashboard

# Смотрим логи
docker logs wazuh-dashboard

# Тестируем (должен вернуть HTML)
curl http://localhost:5602 | head -20
```

**Доступ:** http://ВАШЕ_IP:5602

### Альтернатива: Использовать docker-compose для Dashboard

```bash
# Добавляем Dashboard в существующий docker-compose.yml
cat >> docker-compose.yml << 'EOF'

  wazuh-dashboard:
    image: wazuh/wazuh-dashboard:latest
    container_name: wazuh-dashboard
    network_mode: "host"
    environment:
      - INDEXER_URL=http://localhost:9200
      - INDEXER_USERNAME=admin
      - INDEXER_PASSWORD=admin
      - WAZUH_API_URL=http://localhost:55000
      - WAZUH_API_USERNAME=wazuh
      - WAZUH_API_PASSWORD=wazuh
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5602"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 40s
EOF

# Запускаем Dashboard
docker-compose up -d wazuh-dashboard

# Проверяем логи
docker-compose logs -f wazuh-dashboard
```

---

## ЭТАП 7: Установка Filebeat (опционально, для сбора логов)

```bash
# Устанавливаем Filebeat
sudo apt install -y filebeat

# Создаем конфиг
sudo bash -c 'cat > /etc/filebeat/filebeat.yml << "EOF"
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/ossec/logs/alerts/alerts.json
  multiline.pattern: '^\{'
  multiline.negate: true
  multiline.match: after

output.elasticsearch:
  hosts: ["localhost:9200"]
  index: "wazuh-alerts-%{+yyyy.MM.dd}"
  protocol: "http"

logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
EOF'

# Запускаем
sudo systemctl daemon-reload
sudo systemctl enable filebeat
sudo systemctl start filebeat

# Проверяем статус
sudo systemctl status filebeat
```

---

## ЭТАП 8: Конфигурация Firewall

```bash
# Разрешаем SSH (ВАЖНО!)
sudo ufw allow 22/tcp

# Разрешаем порты Wazuh Manager
sudo ufw allow 1514/tcp
sudo ufw allow 1514/udp
sudo ufw allow 1515/tcp

# Разрешаем порты Elasticsearch
sudo ufw allow 9200/tcp

# Разрешаем порты Kibana и Dashboard
sudo ufw allow 5601/tcp
sudo ufw allow 5602/tcp

# Разрешаем Wazuh API
sudo ufw allow 55000/tcp

# Включаем firewall
sudo ufw enable

# Проверяем статус
sudo ufw status
```

---

## ЭТАП 9: Проверка всех сервисов

```bash
# Проверяем все сервисы на хосте
sudo systemctl status elasticsearch wazuh-manager filebeat

# Проверяем контейнеры Docker
docker ps

# Проверяем слушающие порты
sudo ss -tlnp 2>/dev/null | grep -E "9200|5601|5602|1514|1515|55000"

# Проверяем логи Wazuh Manager
sudo tail -20 /var/ossec/logs/ossec.log

# Проверяем логи Kibana в Docker
docker logs kibana

# Проверяем логи Dashboard в Docker
docker logs wazuh-dashboard
```

---

## ЭТАП 10: Установка Wazuh Agent на клиентских машинах

### На клиентской машине (Linux):

```bash
# Добавляем репозиторий Wazuh
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import -
sudo chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list

# Обновляем и устанавливаем агента
sudo apt update
sudo apt install -y wazuh-agent

# Регистрируем агента (замените IP на адрес вашего Manager)
sudo /var/ossec/bin/agent-auth -m 192.168.1.100 -A agent-linux-01

# Запускаем агента
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent

# Проверяем статус
sudo systemctl status wazuh-agent
```

### На Manager'е для управления агентами:

```bash
# Список всех агентов
sudo /var/ossec/bin/agent_control -l

# Статус конкретного агента (замените ID)
sudo /var/ossec/bin/agent_control -i 001

# Удалить агента
sudo /var/ossec/bin/manage_agents -r AGENT_ID
```

---

## ЭТАП 11: Первый доступ

### Kibana / Wazuh Dashboard
```
http://ВАШЕ_IP:5601
```
- Полнофункциональная аналитика
- Без пароля
- Стабилен и быстро загружается

### Wazuh API
```
https://ВАШЕ_IP:55000
```
- REST API для управления
- Требует аутентификации

---

## Полезные команды Docker

```bash
# Просмотр всех контейнеров
docker ps -a

# Остановить контейнер Kibana
docker stop kibana

# Перезагрузить контейнер
docker restart kibana

# Удалить контейнер
docker rm kibana

# Посмотреть логи
docker logs -f kibana

# Посмотреть последние 50 строк логов
docker logs --tail 50 kibana

# Вход в контейнер (если нужно)
docker exec -it kibana bash

# Очистить все неиспользуемые образы
docker image prune -a

# Просмотр объема дискового пространства Docker
docker system df
```

---

## Полезные команды Wazuh/Elasticsearch

```bash
# Перезагрузить сервис
sudo systemctl restart wazuh-manager

# Посмотреть логи Manager
sudo tail -f /var/ossec/logs/ossec.log

# Проверить конфиг Manager
sudo /var/ossec/bin/wazuh-control validate-config

# Посмотреть индексы Elasticsearch
curl http://localhost:9200/_cat/indices?v

# Удалить старые индексы
curl -X DELETE http://localhost:9200/.wazuh-*
curl -X DELETE http://localhost:9200/.opendistro-*

# Проверить здоровье кластера
curl http://localhost:9200/_cluster/health?pretty

# Посмотреть все алерты
curl http://localhost:9200/wazuh-alerts-*/_search?size=100

# Проверить статус всех сервисов
sudo systemctl status elasticsearch wazuh-manager filebeat
```

---

## Решение типичных проблем

### 1. Elasticsearch не запускается

**Решение:**
```bash
# Создаем директорию для PID
sudo mkdir -p /var/run/elasticsearch
sudo chown elasticsearch:elasticsearch /var/run/elasticsearch
sudo chmod 755 /var/run/elasticsearch

# Проверяем логи
sudo tail -50 /var/log/elasticsearch/elasticsearch.log

# Даем права
sudo chown -R elasticsearch:elasticsearch /var/lib/elasticsearch
sudo chown -R elasticsearch:elasticsearch /var/log/elasticsearch

# Перезапускаем
sudo systemctl restart elasticsearch
```

### 2. Kibana в Docker не может подключиться к Elasticsearch

**Решение:**
```bash
# Проверяем что Elasticsearch работает
curl http://localhost:9200

# Смотрим логи Kibana
docker logs kibana

# Если используется --network host, это должно работать
# Если используется bridge сеть, используйте:
# ELASTICSEARCH_HOSTS=http://host.docker.internal:9200

# Перезагружаем контейнер
docker restart kibana
```

### 3. Порт 5601 уже занят

**Решение:**
```bash
# Проверяем что занимает порт
sudo lsof -i :5601

# Если используется docker-compose, меняем порт:
# ports:
#   - "5602:5601"

# Или убиваем процесс
sudo kill -9 <PID>

# Перезагружаем контейнер
docker restart kibana
```

### 4. Нет данных в Dashboard/Kibana

**Решение:**
```bash
# Проверьте Filebeat
sudo systemctl status filebeat
sudo tail -f /var/log/filebeat/filebeat

# Проверьте логи Manager
sudo tail -50 /var/ossec/logs/ossec.log

# Проверьте индексы
curl http://localhost:9200/_cat/indices?v
```

### 5. Docker контейнер постоянно перезагружается

**Решение:**
```bash
# Смотрим логи
docker logs kibana

# Проверяем статус Elasticsearch
curl http://localhost:9200

# Убеждаемся что используется правильная версия образа
docker inspect kibana | grep Image

# Перезапускаем контейнер с явными параметрами
docker stop kibana
docker rm kibana
docker run -d \
  --name kibana \
  --network host \
  -e ELASTICSEARCH_HOSTS=http://localhost:9200 \
  -e xpack.security.enabled=false \
  docker.elastic.co/kibana/kibana:8.19.6
```

---

## Структура установки

```
Manager (Debian 13):
├── Wazuh Manager (порт 1514, 1515, 55000)
├── Elasticsearch 8.x (порт 9200) - на хосте
├── Kibana 8.x (порт 5601) - в Docker контейнере
└── Filebeat (сбор логов) - на хосте

Agents (любые ОС):
└── Wazuh Agent → подключение к Manager на порту 1514/1515
```

---

## Резюме

✅ **Установлено:**
- Wazuh Manager 4.x (на хосте)
- Elasticsearch 8.x (на хосте)
- Kibana 8.x (в Docker)
- Filebeat (на хосте, опционально)

✅ **Доступно:**
- Kibana: http://IP:5601 (аналитика и логи)
- API: https://IP:55000 (управление)

✅ **Преимущества Docker подхода:**
- Нет проблем с репозиториями и зависимостями
- Легко обновлять версию
- Изоляция от системы
- Легко откатывать изменения

✅ **Готово для:**
- Подключения агентов на других машинах
- Сбора и анализа логов безопасности
- Настройки правил и алертов
- Интеграции с внешними системами