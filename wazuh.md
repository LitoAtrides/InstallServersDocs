# Полная инструкция по установке Wazuh на Debian 13 (Elasticsearch 8.x)

## Предварительные требования

- Debian 13
- 4+ ГБ оперативной памяти
- 50+ ГБ свободного места на диске
- Интернет для загрузки пакетов

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

## ЭТАП 2: Установка Wazuh Manager

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

## ЭТАП 3: Установка Elasticsearch 8.x

**ВАЖНО:** Используем Elasticsearch 8.x (последнюю поддерживаемую версию) для совместимости с Kibana и Wazuh Dashboard.

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
EOF'
```

### Настройка памяти JVM

```bash
# Открываем jvm опции
sudo nano /etc/elasticsearch/jvm.options
```

Найдите строки с `-Xms` и `-Xmx` и измените на:

```
-Xms512m
-Xmx512m
```

Сохраняем (Ctrl+O, Enter, Ctrl+X)

### Запуск Elasticsearch

```bash
# Удаляем старые данные
sudo rm -rf /var/lib/elasticsearch/*
sudo rm -rf /var/log/elasticsearch/*

# Даем права
sudo chown -R elasticsearch:elasticsearch /var/lib/elasticsearch
sudo chown -R elasticsearch:elasticsearch /var/log/elasticsearch
sudo chown -R elasticsearch:elasticsearch /etc/elasticsearch
sudo chmod 755 /var/lib/elasticsearch
sudo chmod 755 /var/log/elasticsearch

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

---

## ЭТАП 4: Установка Kibana 8.x

```bash
# Устанавливаем Kibana
sudo apt install -y kibana

# Создаем конфиг
sudo bash -c 'cat > /etc/kibana/kibana.yml << "EOF"
elasticsearch.hosts: ["http://localhost:9200"]
server.port: 5601
server.host: "0.0.0.0"
server.name: "wazuh-kibana"
xpack.security.enabled: false
EOF'

# Проверяем конфиг
cat /etc/kibana/kibana.yml

# Запускаем
sudo systemctl daemon-reload
sudo systemctl enable kibana
sudo systemctl start kibana

# Ждем 40 секунд на инициализацию
sleep 40

# Проверяем статус
sudo systemctl status kibana

# Тестируем (должен вернуть HTML)
curl http://localhost:5601 | head -20
```

**Доступ:** http://ВАШЕ_IP:5601

---

## ЭТАП 5: Установка Wazuh Dashboard

```bash
# Устанавливаем Dashboard
sudo apt install -y wazuh-dashboard

# Создаем минимальный конфиг
sudo bash -c 'cat > /etc/wazuh-dashboard/opensearch_dashboards.yml << "EOF"
server.port: 5602
server.host: 0.0.0.0
opensearch.hosts: http://localhost:9200
xpack.security.enabled: false
EOF'

# Создаем SSL сертификаты (самоподписанные)
sudo mkdir -p /etc/wazuh-dashboard/certs
sudo openssl req -x509 -newkey rsa:2048 \
  -keyout /etc/wazuh-dashboard/certs/dashboard-key.pem \
  -out /etc/wazuh-dashboard/certs/dashboard.pem \
  -days 365 -nodes \
  -subj "/C=RU/ST=Moscow/L=Moscow/O=Wazuh/CN=wazuh-dashboard"

# Даем права
sudo chown -R wazuh-dashboard:wazuh-dashboard /etc/wazuh-dashboard/certs
sudo chmod 600 /etc/wazuh-dashboard/certs/*

# Запускаем
sudo systemctl daemon-reload
sudo systemctl enable wazuh-dashboard
sudo systemctl start wazuh-dashboard

# Ждем 30 секунд на инициализацию
sleep 30

# Проверяем статус
sudo systemctl status wazuh-dashboard
```

**Доступ:** http://ВАШЕ_IP:5602

---

## ЭТАП 6: Установка Filebeat (опционально, для сбора логов)

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

## ЭТАП 7: Конфигурация Firewall

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

## ЭТАП 8: Проверка всех сервисов

```bash
# Проверяем все сервисы
sudo systemctl status elasticsearch kibana wazuh-dashboard filebeat wazuh-manager

# Проверяем слушающие порты
sudo ss -tlnp 2>/dev/null | grep -E "9200|5601|5602|1514|1515|55000"

# Проверяем логи Wazuh Manager
sudo tail -20 /var/ossec/logs/ossec.log
```

---

## ЭТАП 9: Установка Wazuh Agent на клиентских машинах

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

## ЭТАП 10: Первый доступ

### Kibana
```
http://ВАШЕ_IP:5601
```
- Полнофункциональная аналитика
- Без пароля
- Стабилен и быстро загружается

### Wazuh Dashboard
```
http://ВАШЕ_IP:5602
```
- Специализированный интерфейс Wazuh
- Без пароля
- Полностью совместим с Elasticsearch 8.x

### Wazuh API
```
https://ВАШЕ_IP:55000
```
- REST API для управления
- Требует аутентификации

---

## Полезные команды

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
```

---

## Решение типичных проблем

### 1. Dashboard/Kibana не загружается

**Решение:** Убедитесь что Elasticsearch запущен:
```bash
sudo systemctl status elasticsearch
curl http://localhost:9200
```

### 2. Elasticsearch не запускается

**Решение:**
```bash
# Проверяем логи
sudo tail -50 /var/log/elasticsearch/elasticsearch.log

# Даем права
sudo chown -R elasticsearch:elasticsearch /var/lib/elasticsearch
sudo chown -R elasticsearch:elasticsearch /var/log/elasticsearch

# Перезапускаем
sudo systemctl restart elasticsearch
```

### 3. Агент не подключается к Manager

**Проверьте:**
```bash
# На Manager - список агентов
sudo /var/ossec/bin/agent_control -l

# На агенте - логи
sudo tail -f /var/ossec/logs/agent.log

# Проверьте firewall
sudo ufw status
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

### 5. Ошибка подключения Kibana/Dashboard к Elasticsearch

**Решение:** Убедитесь что в конфигах отключена безопасность:
```bash
# Для Elasticsearch
grep "xpack.security" /etc/elasticsearch/elasticsearch.yml

# Для Kibana
grep "xpack.security" /etc/kibana/kibana.yml

# Для Dashboard
grep "xpack.security" /etc/wazuh-dashboard/opensearch_dashboards.yml
```

Должно быть: `xpack.security.enabled: false`

---

## Структура установки

```
Manager (Debian 13):
├── Wazuh Manager (порт 1514, 1515, 55000)
├── Elasticsearch 8.x (порт 9200)
├── Kibana 8.x (порт 5601)
├── Wazuh Dashboard (порт 5602)
└── Filebeat (сбор логов)

Agents (любые ОС):
└── Wazuh Agent → подключение к Manager на порту 1514/1515
```

---

## Резюме

✅ **Установлено:**
- Wazuh Manager 4.x
- Elasticsearch 8.x (последняя поддерживаемая версия)
- Kibana 8.x
- Wazuh Dashboard
- Filebeat (опционально)

✅ **Доступно:**
- Kibana: http://IP:5601 (аналитика и логи)
- Dashboard: http://IP:5602 (интерфейс Wazuh)
- API: https://IP:55000 (управление)

✅ **Готово для:**
- Подключения агентов на других машинах
- Сбора и анализа логов безопасности
- Настройки правил и алертов
- Интеграции с внешними системами