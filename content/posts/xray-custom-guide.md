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
apt update && apt install golang-go git patch -y
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
# Шаг 3: Патчим Xray

1. Создай файл патча

```diff
cat << 'EOF' > ~/xray_custom.patch
--- freedom.go
+++ freedom.go
@@ -5,6 +5,7 @@
 	"crypto/rand"
 	"io"
 	"time"
+	"fmt"
 
 	"github.com/pires/go-proxyproto"
 	"github.com/xtls/xray-core/common"
@@ -85,8 +86,20 @@
 	ob.Name = "freedom"
 	ob.CanSpliceCopy = 1
 	inbound := session.InboundFromContext(ctx)
-
 	destination := ob.Target
+
+	myJitter := dice.Roll(40) + 10
+	fmt.Printf("[DEBUG] Соединение: %s | Jitter: %dms\n", destination.String(), myJitter)
+	time.Sleep(time.Duration(myJitter) * time.Millisecond)
+
+	myLife := time.Duration(dice.Roll(540)+180) * time.Second
+	fmt.Printf("[DEBUG] Лимит сессии: %v\n", myLife)
+
+	sessionCtx, sessionCancel := context.WithTimeout(ctx, myLife)
+	defer sessionCancel()
+	ctx = sessionCtx
 	origTargetAddr := ob.OriginalTarget.Address
 	if origTargetAddr == nil {
 		origTargetAddr = ob.Target.Address
@@ -165,7 +178,8 @@
 	}
 
 	plcy := h.policy()
-	ctx, cancel := context.WithCancel(ctx)
+	var cancel context.CancelFunc
+	ctx, cancel = context.WithCancel(ctx)
 	timer := signal.CancelAfterInactivity(ctx, func() {
 		cancel()
 		if newCancel != nil {
EOF
```

2. Перейди в папку, где лежит файл freedom.go

```bash
cd ~/xray-src/proxy/freedom/
```

3. Примени изменения из файла патча

```bash
patch freedom.go < ~/xray_custom.patch
```

&nbsp;
# Шаг 4: Компиляция
Собери кастомный файл из измененных исходников:

```bash
cd ~/xray-src/main
```

```bash
go build -v -o xray-custom
```

&nbsp;
# Шаг 5: Активация бинарника
Замени стандартный Xray кастомным и перезапускаем панель:

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