# iptables
## Домашнее задание
Сценарии iptables

- реализовать knocking port
- centralRouter может попасть на ssh inetrRouter через knock скрипт пример в материалах
- добавить inetRouter2, который виден(маршрутизируется (host-only тип сети для виртуалки)) с хоста или форвардится порт через локалхост
- запустить nginx на centralServer
- пробросить 80й порт на inetRouter2 8080
- дефолт в инет оставить через inetRouter
- реализовать проход на 80й порт без маскарадинга
## Выполнение

### Port Knocking
Реализовано средствами iptables:
````
# TRAFFIC chain for Port Knocking. The correct port sequence in this example is  8881 -> 7777 -> 9991; any other sequence will drop the traffic
            sudo iptables -N TRAFFIC
            sudo iptables -N SSH-INPUT
            sudo iptables -N SSH-INPUTTWO
            sudo iptables -A INPUT -j TRAFFIC
            sudo iptables -A TRAFFIC -p icmp --icmp-type any -j ACCEPT
            sudo iptables -A TRAFFIC -m state --state ESTABLISHED,RELATED -j ACCEPT
            sudo iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp --dport 22 -m recent --rcheck --seconds 30 --name SSH2 -j ACCEPT
            sudo iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp -m recent --name SSH2 --remove -j DROP
            sudo iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp --dport 9991 -m recent --rcheck --name SSH1 -j SSH-INPUTTWO
            sudo iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp -m recent --name SSH1 --remove -j DROP
            sudo iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp --dport 7777 -m recent --rcheck --name SSH0 -j SSH-INPUT
            sudo iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp -m recent --name SSH0 --remove -j DROP
            sudo iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp --dport 8881 -m recent --name SSH0 --set -j DROP
            sudo iptables -A SSH-INPUT -m recent --name SSH1 --set -j DROP
            sudo iptables -A SSH-INPUTTWO -m recent --name SSH2 --set -j DROP
            sudo iptables -A TRAFFIC -j DROP
            # COMMIT
  ````
  Запускаем проект:
  ````
  vagrant up
  ````
  Подключаемся по ssh к inetRouter:
  ````
san4ez@edukation:~/IPTABLES$ vagrant ssh centralRouter
[vagrant@centralRouter ~]$ ssh 192.168.255.1 -l vagrant
````
Видим, что просто так на ssh подключится не получается, пробуем простучать порты на inetRouter:
````
[vagrant@centralRouter ~]$ knock 192.168.255.1 8881 7777 9991 -d 1000
[vagrant@centralRouter ~]$ ssh 192.168.255.1 -l vagrant
The authenticity of host '192.168.255.1 (192.168.255.1)' can't be established.
ECDSA key fingerprint is SHA256:m4cWBCDoknHFjNPsl5/8IsJZHQsubq+BpJ527mVJQeY.
ECDSA key fingerprint is MD5:d6:96:40:15:ff:b0:f9:d0:8f:ce:90:b0:14:11:f4:ae.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.255.1' (ECDSA) to the list of known hosts.
vagrant@192.168.255.1's password:
````
После простукивания, подключение прошло успешно, port knocking работает.

### Port forwarding 
 # Форвардинг
 ````
 sudo bash -c 'echo "net.ipv4.conf.all.forwarding=1" >> /etc/sysctl.conf'; sudo sysctl -p
 ````
 - Установка софта для того чтобы можно было сохранить правила iptables после перезагрузки
 ````
 sudo yum install -y iptables-services traceroute; sudo systemctl enable iptables && sudo systemctl start iptables;
 ````
 - Очистка правил iptables
  ````sudo iptables -P INPUT ACCEPT
      sudo iptables -P FORWARD ACCEPT
      sudo iptables -P OUTPUT ACCEPT
      sudo iptables -t nat -F
      sudo iptables -t mangle -F
      sudo iptables -F
      sudo iptables -X
 ````
  - Первое правило: для переадресации пакетов с интерфейса eth0 порта 8080 на хост с веб-сервером nginx 192.168.0.2 порт 80
  - Второе правило: после применения правил маршрутизации к пакету изменяем адрес назначения в пакете, т.е. говорит пакету вернуться на адрес 192.168.255.2
 ````
      sudo iptables -t nat -A PREROUTING -i eth0 -p tcp -m tcp --dport 8080 -j DNAT --to-destination 192.168.0.2:80
      sudo iptables -t nat -A POSTROUTING --destination 192.168.0.2/32 -j SNAT --to-source 192.168.255.2
      sudo service iptables save
 ````
 
  - Убираем маршрут по умолчанию для eth0
 ````
    echo "DEFROUTE=no" >> /etc/sysconfig/network-scripts/ifcfg-eth0
 ````
   - Маршрут по умолчанию на inetRouter (eth1)
 ````
     echo "GATEWAY=192.168.255.1" >> /etc/sysconfig/network-scripts/ifcfg-eth1
 ````
   - Указываем где искать и куда отправлять пакеты для 192.168.0.0/16
 ````
     sudo bash -c 'echo "192.168.0.0/16 via 192.168.255.3 dev eth1" > /etc/sysconfig/network-scripts/route-eth1';
     systemctl restart network
     sudo echo restart network complete!
     sudo reboot            
     sudo iptables -L -v -n
     sudo echo inetRouter2 
     SHELL
````
### В результате если через curl обратиться на localhost:
````
san4ez@edukation:~/IPTABLES$ curl -I localhost:1234
HTTP/1.1 200 OK
Server: nginx/1.20.1
Date: Tue, 27 Jul 2021 10:40:57 GMT
Content-Type: text/html
Content-Length: 4833
Last-Modified: Fri, 16 May 2014 15:12:48 GMT
Connection: keep-alive
ETag: "53762af0-12e1"
Accept-Ranges: bytes
````
### В заключении Default Router:
````
san4ez@edukation:~/IPTABLES$ vagrant ssh inetRouter2
[vagrant@inetRouter2 ~]$ traceroute otus.ru
traceroute to otus.ru (104.26.5.108), 30 hops max, 60 byte packets
 1  gateway (192.168.255.1)  1.769 ms  1.619 ms  2.180 ms
 2  * * *
 3  * * *
 4  104.26.5.108 (104.26.5.108)  106.970 ms  104.751 ms  109.768 ms

````
