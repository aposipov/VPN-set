## Hosting
Критерии по выбору локации для сервера:
- физическая удаленность от вашего местоположения (чем меньше тем лучше);
- страна с более менее "свободным" интернетом;  
- не помешает проверка на минимальный ping;
  
Для Москвы и области отлично подойдет сервер в Германии. Я использую вот этот хостинг для сервера VPN,  
ссылка реферальная https://gcorelabs.com/ru/hosting/?from=9826919 поэтому заранее благодарен за ее использование.
## WireGuard setup

Обновляем сервер:
```
apt update && apt upgrade -y
```

Ставим wireguard:
```
apt install -y wireguard
```

Генерим ключи сервера:
```
wg genkey | tee /etc/wireguard/privatekey | wg pubkey | tee /etc/wireguard/publickey
```

Проставляем права на приватный ключ:
```
chmod 600 /etc/wireguard/privatekey
```
Создаём конфиг сервера:
```
vim /etc/wireguard/wg0.conf
```
Копируем в файл:
```
[Interface]  
PrivateKey = <privatekey>
Address = 10.0.0.1/24
ListenPort = 51830
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```
Интерфейс `eth0` заменяем на свой, посмотреть можно следующей командой:
```
ifconfig
```
нужен тот, который соответствует вашему IP адресу сервера  
Вставляем вместо `privatekey` содержимое файла `/etc/wireguard/privatekey`  
Настраиваем IP форвардинг:
```
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```
Включаем systemd демон с wireguard:
```
systemctl enable wg-quick@wg0.service
systemctl start wg-quick@wg0.service
systemctl status wg-quick@wg0.service
```  
Создаём ключи клиента:
```
wg genkey | tee /etc/wireguard/client_privatekey | wg pubkey | tee /etc/wireguard/client_publickey
```
Добавляем в конфиг сервера клиента:
```
vim /etc/wireguard/wg0.conf
```
```
[Peer]
PublicKey = <client_publickey>
AllowedIPs = 10.0.0.2/32
```
Вместо `client_publickey`  — заменяем на содержимое файла `/etc/wireguard/client_publickey`

Перезагружаем systemd сервис с wireguard:
```
systemctl restart wg-quick@wg0
systemctl status wg-quick@wg0
```
На локальной машине (например, на ноутбуке) создаём текстовый файл с конфигом клиента:
```
vim clientn_wb.conf
```
```
[Interface]
PrivateKey = <CLIENT-PRIVATE-KEY>
Address = 10.0.0.2/32
DNS = 8.8.8.8

[Peer]
PublicKey = <SERVER-PUBKEY>
Endpoint = <SERVER-IP>:51830
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 20
```
Здесь `CLIENT-PRIVATE-KEY` заменяем на приватный ключ клиента, то есть содержимое 
файла `/etc/wireguard/client_privatekey` на сервере.  
`SERVER-PUBKEY` заменяем на публичный ключ сервера, то есть на содержимое 
файла `/etc/wireguard/publickey` на сервере. `SERVER-IP` заменяем на IP сервера. 

Этот файл открываем в Wireguard клиенте (есть для всех операционных систем, в том 
числе мобильных) — и жмем в клиенте кнопку подключения.

посмореть текущую конфигурацию и статистику
```
wg show
```
включить клиента на unix системе
```
wg-quick down client_conf
wg-quick up client_conf
```
создать qr код для мобильного устройства можно с помощьюприложения `qrencode`
```
qrencode -t ansiutf8 < mobile.conf
```

## Usefull links

[Официальный сайт WireGuard](https://www.wireguard.com/quickstart/#command-line-interface)  
[Видео по оплате и настройки хостинга](https://www.youtube.com/watch?v=XvbY-xY5dWY)    
[WireGuard настраивал по этому видео, за основу взял его мануал с ТГ](https://www.youtube.com/watch?v=5Aql0V-ta8A&t=223s)   

## Wireguard in Docker
скачивание и установка Docker скрипт с официального сайта
```
sudo curl -fsSL https://get.docker.com/ -o get-docker.sh
```
```
sudo chmod +x ./get-docker.sh && ./get-docker.sh
```
добавляем в группу docker текущего пользователя
```
sudo usermod -aG docker $USER
```
я воспользовался готовым образом от [linuxserver.io](https://github.com/linuxserver/docker-wireguard) еще вариант альтернативного образа от [cmulk](https://github.com/cmulk/wireguard-docker/tree/main)
```
docker pull linuxserver/wireguard
```
```
docker run -d \
  --name=wireguard \
  --cap-add=NET_ADMIN \
  -e PUID=1000 \
  -e PGID=1000 \
  -p 51820:51820/udp \
  -v /<path to config>:/config \
  --restart unless-stopped \
  linuxserver/wireguard:latest
```
`-p 51820:51820/udp` укажите ваш порт из конфигурации и внешний который открыт на сервере  
`-v /<path to config>:/config` путь до директории со всеми ключами и конфигурацией на сервере
  
чтобы скопировать ваш конфиг воспользуйтесь командой
```
scp  -P 4242 -r ./<your local dir> <username>@<ip addres>:/<path server dir>
```
