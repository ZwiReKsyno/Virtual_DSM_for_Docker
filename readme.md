<h1 align="center">Виртуальный DSM для Docker<br />
<div align="center">
<img src="https://github.com/vdsm/virtual-dsm/raw/master/.github/screen.jpg" title="Screenshot" style="max-width:100%;" width="432" />


## Функции

- Мультиплатформенность
- KVM-ускорение
- Сквозная передача графического процессора
- Плавное завершение работы
- Поддерживаются обновления
 
## Применение

С помощью`docker-compose.yml`

```yaml
version: "3"
services:
  dsm:
    container_name: dsm
    image: vdsm/virtual-dsm:latest
    environment:
      DISK_SIZE: "16G"
    devices:
      - /dev/kvm
      - /dev/vhost-net
    cap_add:
      - NET_ADMIN
    ports:
      - 5000:5000
    volumes:
      - /opt/dsm:/storage
    restart: on-failure
    stop_grace_period: 1m
```

С помощью `docker run`

```bash
docker run -it --rm -p 5000:5000 --device=/dev/kvm --cap-add NET_ADMIN --stop-timeout 60 vdsm/virtual-dsm:latest
```

## Часто задаваемые вопросы

  * ### Как изменить размер виртуального диска?

    Чтобы увеличить размер по умолчанию (16 ГБ), найдите этот DISK_SIZEпараметр в файле создания и измените его на желаемую емкость:

    ```yaml
    environment:
      DISK_SIZE: "256G"
    ```
    
    Это также можно использовать для изменения размера существующего диска до большей емкости без потери данных.

  * ### Как изменить расположение виртуального диска?

    Чтобы изменить расположение виртуального диска по сравнению с томом Docker по умолчанию, включите следующую привязку в свой файл создания файла:

    ```yaml
    volumes:
      - /home/user/data:/storage
    ```

   Замените путь примера /home/user/dataна нужную папку хранения.

  * ### Как изменить пространство, зарезервированное виртуальным диском?

  По умолчанию все дисковое пространство зарезервировано заранее. Чтобы создать расширяемый диск, на котором резервируется только то пространство, которое фактически используется, добавьте следующую переменную среды:

    ```yaml
    environment:
      ALLOCATE: "N"
    ```

    Имейте в виду, что это не повлияет ни на один из существующих дисков, а относится только к вновь созданным дискам.

  * ### Как увеличить объем процессора или оперативной памяти?

   По умолчанию контейнеру выделено одно ядро ​​и 512 МБ оперативной памяти. Чтобы увеличить это значение, добавьте следующие переменные среды:

    ```yaml
    environment:
      CPU_CORES: "4"
      RAM_SIZE: "2048M"
    ```

  * ### Как проверить, поддерживает ли моя система KVM?
     - KVM программное решение, обеспечивающее виртуализацию в среде Linux 

    Чтобы проверить, поддерживает ли ваша система KVM, выполните следующие команды:

    ```bash
    sudo apt install cpu-checker
    sudo kvm-ok
    ```

   Если вы получаете сообщение об ошибке, kvm-okуказывающее, что ускорение KVM невозможно использовать, проверьте настройки BIOS.

  * ### Как назначить контейнеру индивидуальный IP-адрес?

   По умолчанию контейнер использует мостовую сеть, которая использует общий IP-адрес с хостом.

    Если вы хотите назначить контейнеру индивидуальный IP-адрес, вы можете создать сеть macvlan следующим образом:

    ```bash
    docker network create -d macvlan \
        --subnet=192.168.0.0/24 \
        --gateway=192.168.0.1 \
        --ip-range=192.168.0.100/28 \
        -o parent=eth0 vdsm
    ```
    
   Обязательно измените эти значения в соответствии с вашей локальной подсетью.

   После создания сети измените файл Compose, чтобы он выглядел следующим образом:

    ```yaml
    services:
      dsm:
        container_name: dsm
        ..<snip>..
        networks:
          vdsm:
            ipv4_address: 192.168.0.100

    networks:
      vdsm:
        external: true
    ```
   
   Дополнительным преимуществом этого подхода является то, что вам больше не придется выполнять сопоставление портов, поскольку все порты будут открыты по умолчанию.

   Обратите внимание, что этот IP-адрес не будет доступен с хоста Docker из-за конструкции macvlan, которая не разрешает связь между ними. Если это вызывает беспокойство, в качестве обходного пути вам необходимо создать  [second macvlan](https://blog.oddbit.com/post/2018-03-12-using-docker-macvlan-networks/#host-access) as a workaround.

  * ### Как контейнер может получить IP-адрес от моего маршрутизатора?

   После настройки контейнера для macvlan (см. выше) DSM может стать частью вашей домашней сети, запросив IP-адрес у вашего маршрутизатора, как и другие ваши устройства.

   Чтобы включить эту функцию, добавьте следующие строки в файл компоновки:

    ```yaml
    environment:
      DHCP: "Y"
    devices:
      - /dev/vhost-net
    device_cgroup_rules:
      - 'c *:* rwm'
    ```

    Обратите внимание: даже если вам не нужен DHCP, все равно рекомендуется включить эту функцию, поскольку она предотвращает проблемы с NAT и повышает производительность за счет использования macvtapинтерфейса.

  * ### Как установить определенную версию vDSM?

   По умолчанию будет установлена ​​версия 7.2.1, но если вы предпочитаете более старую версию, вы можете добавить ее URL-адрес для загрузки в свой файл создания следующим образом:

    ```yaml
    environment:
      URL: "https://global.synologydownload.com/download/DSM/release/7.0.1/42218/DSM_VirtualDSM_42218.pat"
    ```

   С помощью этого метода можно даже переключаться между различными версиями, сохраняя при этом все данные вашего файла.

  * ### Как мне пройти через мой графический процессор?

    Чтобы передать ваш графический процессор Intel, добавьте следующие строки в файл компоновки:

    ```yaml
    environment:
      GPU: "Y"
    devices:
      - /dev/dri
    ```

    Это можно использовать, например, для включения функции распознавания лиц в Synology Photos.
    
  * ### Каковы различия по сравнению со стандартным DSM?

    Есть только два незначительных отличия: пакет Virtual Machine Manager недоступен, а Surveillance Station не будет включать никаких бесплатных лицензий.
    
  * ### Отказ от ответственности
  
     Запускайте этот контейнер только на оборудовании Synology, любое другое использование не разрешено лицензионным соглашением. Названия продуктов, логотипы, бренды и другие товарные знаки, упомянутые в этом проекте, являются собственностью соответствующих владельцев товарных знаков. Этот проект не связан, не спонсируется и не поддерживается Synology, Inc.
