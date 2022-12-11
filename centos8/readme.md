Задание 1

1) Обновить ядро ОС из репозитория ELRepo
2) Создать Vagrant box c помощью Packer
3) Загрузить Vagrant box в Vagrant Cloud

В директории /centos8 находится Vagrantfile, в котором сконфигурирована
виртуальная машина с обновлённым ядром.

Конфиг packer'a: /centos8/centos8.json
Файл автоматической настройки: /centos8/http/ks.cfg
Скрипты, запускаемые после установки: /centos8/
  stage-1-kernel-update.sh -- обновление ядра
  stage-1-kernel-update.sh -- очистка системы для уменьшения объёма
