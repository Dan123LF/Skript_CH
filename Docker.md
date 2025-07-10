#Развёртывание ClickHouse в Docker — пошаговое руководство

Введение
ClickHouse — это высокопроизводительная аналитическая СУБД, построенная на колоночном хранении данных. Docker позволяет быстро и удобно развернуть ClickHouse в изолированном контейнере, что упрощает установку, масштабирование и управление.

Шаг 1. Установка Docker
Перед началом убедитесь, что Docker установлен на вашем сервере или локальной машине.

Проверка установки:

bash
docker --version
Если Docker не установлен, следуйте официальной инструкции по установке Docker для вашей ОС:
https://docs.docker.com/get-docker/

Шаг 2. Загрузка официального образа ClickHouse
Скачайте официальный образ ClickHouse с Docker Hub:

bash
docker pull clickhouse/clickhouse-server
Шаг 3. Создание директорий для данных и конфигурации (рекомендуется)
Чтобы данные ClickHouse сохранялись между перезапусками контейнера, создайте локальные папки для хранения данных и логов:

bash
sudo mkdir -p /opt/clickhouse/{data,etc,log}
Шаг 4. Запуск контейнера ClickHouse
Запустите контейнер с монтированием директорий и настройкой лимитов:

bash
docker run -d --name clickhouse-server \
  --ulimit nofile=262144:262144 \
  -v /opt/clickhouse/data:/var/lib/clickhouse \
  -v /opt/clickhouse/log:/var/log/clickhouse-server \
  -v /opt/clickhouse/etc:/etc/clickhouse-server \
  -p 8123:8123 -p 9000:9000 \
  clickhouse/clickhouse-server
--ulimit nofile=262144:262144 — увеличивает лимит открытых файлов для производительности.

-v — монтирует локальные папки для сохранения данных и логов.

-p — пробрасывает порты: 8123 для HTTP интерфейса, 9000 для TCP клиента.

Шаг 5. Проверка работы контейнера
Убедитесь, что контейнер запущен:

bash
docker ps
Вы должны увидеть контейнер с именем clickhouse-server в списке.

Шаг 6. Подключение к ClickHouse внутри контейнера
Для подключения к серверу используйте встроенный клиент:

bash
docker exec -it clickhouse-server clickhouse-client
В клиенте можно выполнить тестовый запрос:

sql
SELECT 'Hello, ClickHouse!';
Для выхода из клиента:

sql
exit;
Шаг 7. Настройка сети и безопасности (опционально)
По умолчанию ClickHouse слушает все интерфейсы (0.0.0.0), но убедитесь, что доступ к портам 8123 и 9000 ограничен файрволом.

Для настройки файрвола на Ubuntu с ufw:

bash
sudo ufw allow ssh
sudo ufw allow 9000/tcp
sudo ufw allow 8123/tcp
sudo ufw enable
sudo ufw status
Для продакшен-среды рекомендуем разрешать доступ только с доверенных IP.

Шаг 8. Создание volume и сети Docker (для продвинутого использования)
Для удобства управления данными и сетями создайте volume и сеть:

bash
docker volume create clickhouse_data
docker network create ch_net
Запуск контейнера с использованием volume и сети:

bash
docker run -d --name clickhouse-server \
  --ulimit nofile=262144:262144 \
  -v clickhouse_data:/var/lib/clickhouse \
  --network ch_net \
  -p 8123:8123 -p 9000:9000 \
  clickhouse/clickhouse-server
Шаг 9. Остановка и удаление контейнера
Чтобы остановить контейнер:

bash
docker stop clickhouse-server
Чтобы удалить контейнер:

bash
docker rm clickhouse-server
Дополнительные рекомендации
Для кластеризации и масштабирования изучите Docker Compose или Docker Swarm.

Используйте внешние конфигурационные файлы для тонкой настройки ClickHouse.

Регулярно создавайте бэкапы данных.

Для автоматизации развертывания можно использовать Ansible.

Итог
Вы успешно развернули ClickHouse в Docker-контейнере, обеспечив сохранность данных и доступ через стандартные порты. Теперь можно создавать базы данных, таблицы и выполнять аналитические запросы.
