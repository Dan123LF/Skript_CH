Установка и настройка PIX BI в WSL с использованием snap и nginx
Данная инструкция описывает процесс установки, настройки и запуска веб-приложения PIX BI на системе WSL (Windows Subsystem for Linux) с использованием snap, .NET, systemd сервисов и обратного прокси nginx.
Шаг 1. Установка и подготовка окружения
•	Убедитесь, что у вас установлен и обновлен snap.
•	Установлен .NET runtime через snap и скопирован в varwwwpixbi исполняемый файл pix-bi.dll и все сопутствующие файлы.
•	Пользователь www-data существует и у него домашний каталог в homewww-data.
Шаг 2. Настройка systemd сервиса для запуска приложения
Создайте или отредактируйте файл сервиса etcsystemdsystempixbi.service со следующим содержимым
text
[Unit]
Description=ASP .NET Web Application PIX BI
After=network.target

[Service]
User=www-data
WorkingDirectory=varwwwpixbi
ExecStart=snapbindotnet varwwwpixbipix-bi.dll
Restart=always
RestartSec=10
SyslogIdentifier=pixbi

# Важно указать домашний каталог для Snap-приложения
Environment=HOME=homewww-data

# Дополнительные переменные окружения
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=LANG=ru_RU.UTF-8
Environment=XDG_CONFIG_HOME=tmp
Environment=XDG_CACHE_HOME=tmp

[Install]
WantedBy=multi-user.target
Объяснение и рекомендации
•	Параметр Environment=HOME=homewww-data критичен для корректной работы snap-пакетов с пользователями вне home по умолчанию.
•	WorkingDirectory должен указывать на каталог с приложением, чтобы dotnet корректно находил настройки и ресурсы.
•	Сервис запускается от пользователя www-data для ограничения прав и безопасности.
Шаг 3. Настройка nginx как обратного прокси с поддержкой WebSocket и SSL
Пример файла etcnginxconf.dpixbi.conf
text
server {
  listen 80;
  server_name localhost pixbi.yourcompany.local;
  return 301 https$host$request_uri;
}

server {
  listen 443 ssl;
  server_name pixbi.yourcompany.local;

  ssl_certificate etcnginxsslpixbi.crt;
  ssl_certificate_key etcnginxsslpixbi.key;
  ssl_protocols TLSv1.2 TLSv1.3;

  access_log varlognginxpixbi.access.log;
  error_log varlognginxpixbi.error.log;

  location  {
    proxy_pass http127.0.0.15000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection upgrade;
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;

    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }

  location ws {
    proxy_pass http127.0.0.15000ws;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection Upgrade;
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
    proxy_read_timeout 36000s;
    proxy_send_timeout 36000s;
  }
}
Объяснение
•	Первый сервер слушает HTTP на 80 порт и перенаправляет все запросы на HTTPS.
•	Второй сервер с SSL сертификатами проксирует запросы к backend приложению.
•	Локация ws специально настроена на проксирование WebSocket (wss), с правильными заголовками Upgrade и Connection.
•	Используйте только TLSv1.2 и TLSv1.3 для безопасности.
Шаг 4. Запуск и тестирование
1.	Запустите или перезапустите сервис pixbi
bash
sudo systemctl daemon-reload
sudo systemctl restart pixbi.service
sudo systemctl status pixbi.service
2.	Проверьте, что backend слушает на порту 5000
bash
sudo ss -tlnp  grep 5000
3.	Проверьте работу nginx
bash
sudo nginx -t
sudo systemctl reload nginx
4.	В браузере откройте
text
httpspixbi.yourcompany.local
Важно В файле hosts на Windows и WSL должно быть прописано
text
127.0.0.1 pixbi.yourcompany.local
Особенности запуска приложения в WSL
•	В WSL systemd может работать с ограничениями, из-за чего сервисы на snap могут не стартовать корректно.
•	Рекомендуемый способ запуска — вручную от пользователя www-data с правильными переменными окружения и из нужного каталога
bash
sudo -u www-data snapbindotnet varwwwpixbipix-bi.dll
•	Такой запуск обходит сложности с systemd в WSL.
Автоматический запуск при старте компьютера (в WSL)
Если systemd нестабилен, рекомендуем создать скрипт запуска, который нужно будет запускать вручную или через автозагрузку пользователя.
Пример скрипта start-pixbi.sh
bash
#!binbash
cd varwwwpixbi
sudo -u www-data snapbindotnet .pix-bi.dll &
Команды для ручного запуска после загрузки WSL
bash
cd varwwwpixbi
sudo -u www-data snapbindotnet .pix-bi.dll
Или если хотите в фоне
bash
sudo -u www-data snapbindotnet varwwwpixbipix-bi.dll &
Краткий список команд для запуска PIX BI после загрузки компьютера
bash
# Перейти в директорию приложения
cd varwwwpixbi

# Запустить приложение под пользователем www-data
sudo -u www-data snapbindotnet .pix-bi.dll

# (Опционально) Чтобы запустить в фоне, добавьте &
sudo -u www-data snapbindotnet .pix-bi.dll &

# Проверить статус сервиса pixbi.service (если используете systemd)
sudo systemctl status pixbi.service

# Проверить, слушает ли backend 5000 порт
sudo ss -tlnp  grep 5000

# Проверить конфигурацию и перезапустить nginx (если меняли настройки)
sudo nginx -t
sudo systemctl reload nginx
Если хотите автоматизировать запуск с systemd, убедитесь, что в unit добавлена переменная окружения HOME и сервис работает стабильно в WSL.

