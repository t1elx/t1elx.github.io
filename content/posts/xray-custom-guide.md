+++
title = "Инструкция по развертыванию Custom Xray"
date = "2026-01-16T22:00:00+05:00"
draft = false
author = "t1elx"
robotsNoIndex = true


[build]
  list = 'never'      # Не показывать в списке постов
  render = 'always'   # Но разрешить открывать по прямой ссылке
  publishResources = false
+++

> **Примечание:** Эта страница скрыта из общего списка и доступна только по прямой ссылке.


&nbsp;
# Шаг 1: Подготовка нового сервера.
Установи панель управления 3x-ui (если не установлена) и язык Go для сборки:

```bash
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
```

```bash
apt update && apt install golang-go git -y
```

&nbsp;
# Шаг 2: Загрузка исходников Xray

```bash
cd ~
```

```bash
git clone https://github.com/XTLS/Xray-core.git xray-src
```

&nbsp;
# Шаг 3: Замена файла прошивки


Открой файл через `nano` и замени содержимое:

```bash
nano /root/xray-src/proxy/freedom/freedom.go
```

> За файлом стучись в telegram: @t1elx

&nbsp;
# Шаг 4: Компиляция
Собираем кастомный файл из измененных исходников:

```bash
cd ~/xray-src/main
```

```bash
go build -v -o xray-custom
```

&nbsp;
# Шаг 5: Активация бинарника
Заменяем стандартный Xray кастомным и перезапускаем панель:

```bash
systemctl stop x-ui
```

```bash
cp ~/xray-src/main/xray-custom /usr/local/x-ui/bin/xray-linux-amd64
```

```bash
chmod +x /usr/local/x-ui/bin/xray-linux-amd64
```

```bash
systemctl start x-ui
```
> **ВАЖНО:** Никогда не нажимай кнопку "Update Xray" в веб-интерфейсе панели. Это удалит твой кастомный бинарник.

&nbsp;
# Шаг 6: Проверка работы (Режим отладки)
Чтобы увидеть [DEBUG] логи, останови сервис и запусти Xray вручную:

```bash
systemctl stop x-ui
```
```bash
/usr/local/x-ui/bin/xray-linux-amd64 -c /usr/local/x-ui/bin/config.json
```
После этого подключись к VPN и открой пару сайтов.

Когда убедишься, что логи идут, нажми Ctrl + C и верни сервис в работу:
```bash
systemctl start x-ui 
```

> ГОТОВО!