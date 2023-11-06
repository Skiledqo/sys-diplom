# Дипломная работа по профессии «Системный администратор» - Иван Серебренников

## Задача

Ключевая задача — разработать отказоустойчивую инфраструктуру для сайта, включающую мониторинг, сбор логов и резервное копирование основных данных. Инфраструктура должна размещаться в Yandex Cloud и отвечать минимальным стандартам безопасности: запрещается выкладывать токен от облака в git.

## Инфраструктура

Для решения задачи было развернуто 6 ВМ (2 для веб-серверов, 1 - Elasticsearch, 1 - Zabbix_server, 1 - Kibana, 1- Bastion Host), а также виртуальная сеть для организации взаимодействия машин и сервисов.

Для развертки инфраструктуры использовался Terraform. 
Файл main.tf

```yaml
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
  required_version = ">= 0.13"
}

provider "yandex" {
  service_account_key_file = "/home/ivan/key.json"
  cloud_id                = "####################"
  folder_id               = "####################"
  zone                    = "ru-central1-b"
}

resource "yandex_vpc_network" "netology" {
  name = "netology"
}

resource "yandex_vpc_subnet" "bastion" {
  v4_cidr_blocks = ["172.16.50.0/24"]
  zone           = "ru-central1-b"
  network_id     = "${yandex_vpc_network.netology1.id}"
}


resource "yandex_vpc_subnet" "private-central-a" {
  v4_cidr_blocks = ["172.16.27.0/24"]
  zone           = "ru-central1-a"
  network_id     = "${yandex_vpc_network.netology1.id}"
}

resource "yandex_vpc_subnet" "private-central-b" {
  v4_cidr_blocks = ["172.16.17.0/24"]
  zone           = "ru-central1-b"
  network_id     = "${yandex_vpc_network.netology1.id}"
}

resource "yandex_vpc_subnet" "private-central-c" {
  v4_cidr_blocks = ["172.16.37.0/24"]
  zone           = "ru-central1-c"
  network_id     = "${yandex_vpc_network.netology1.id}"
}

resource "yandex_vpc_subnet" "public-central-a" {
  v4_cidr_blocks = ["172.16.28.0/24"]
  zone           = "ru-central1-a"
  network_id     = "${yandex_vpc_network.netology1.id}"
}

resource "yandex_vpc_subnet" "public-central-b" {
  v4_cidr_blocks = ["172.16.18.0/24"]
  zone           = "ru-central1-b"
  network_id     = "${yandex_vpc_network.netology1.id}"
}

resource "yandex_vpc_subnet" "public-central-c" {
  v4_cidr_blocks = ["172.16.38.0/24"]
  zone           = "ru-central1-c"
  network_id     = "${yandex_vpc_network.netology1.id}"
}


resource "yandex_compute_instance" "elasticsearch" {
  name = "elasticsearch"
  zone = "ru-central1-b"
  platform_id = "standard-v3"
  resources {
    cores = 2
    core_fraction = 100
    memory = 8
    
  }
  boot_disk {
    initialize_params {
      image_id = "fd8tr2sle07nhtq0idhb"
      size = 29
    }
  }
  network_interface {
      subnet_id = yandex_vpc_subnet.private-central-b.id
      nat=false
     
  }
  metadata = {
    user-data = "${file("/home/ivan/terraform/meta-elastic.txt")}"
  }
  
}

resource "yandex_compute_instance" "bastion-nat" {
  name = "bastion-nat"
  zone = "ru-central1-b"
  platform_id = "standard-v3"
  resources {
    cores = 2
    core_fraction = 100
    memory = 4

  }
  boot_disk {
    initialize_params {
      image_id = "fd8tr2sle07nhtq0idhb"
      size = 13
    }
  }
  network_interface {
      subnet_id = yandex_vpc_subnet.bastion.id
      nat=true

  }
  metadata = {
    user-data = "${file("/home/ivan/terraform/meta-bastion.txt")}"
  }

}

resource "yandex_compute_instance" "web-server1" {
  name = "web-server1"
  zone = "ru-central1-b"
  platform_id = "standard-v3"
  resources {
    cores = 2
    core_fraction = 20
    memory = 2

  }
  boot_disk {
    initialize_params {
      image_id = "fd8tr2sle07nhtq0idhb"
      size = 13
    }
  }
  network_interface {
      subnet_id = yandex_vpc_subnet.private-central-b.id
      nat=false

  }
  metadata = {
    user-data = "${file("/home/ivan/terraform/meta-web-server1.txt")}"
  }

}

resource "yandex_compute_instance" "web-server2" {
  name = "web-server2"
  zone = "ru-central1-a"
  platform_id = "standard-v3"
  resources {
    cores = 2
    core_fraction = 20
    memory = 2

  }
  boot_disk {
    initialize_params {
      image_id = "fd8tr2sle07nhtq0idhb"
      size = 13
    }
  }
  network_interface {
      subnet_id = yandex_vpc_subnet.private-central-a.id
      nat=false

  }
  metadata = {
    user-data = "${file("/home/ivan/terraform/meta-web-server2.txt")}"
  }

}

resource "yandex_compute_instance" "zabbix-server" {
  name = "zabbix-server"
  zone = "ru-central1-b"
  platform_id = "standard-v3"
  resources {
    cores = 2
    core_fraction = 100
    memory = 4

  }
  boot_disk {
    initialize_params {
      image_id = "fd8tr2sle07nhtq0idhb"
      size = 39
    }
  }
  network_interface {
      subnet_id = yandex_vpc_subnet.public-central-b.id
      nat=true

  }
  metadata = {
    user-data = "${file("/home/ivan/terraform/meta-zabbix.txt")}"
  }

}

resource "yandex_compute_instance" "kibana" {
  name = "kibana"
  zone = "ru-central1-b"
  platform_id = "standard-v3"
  resources {
    cores = 2
    core_fraction = 20
    memory = 2

  }
  boot_disk {
    initialize_params {
      image_id = "fd8tr2sle07nhtq0idhb"
      size = 13
    }
  }
  network_interface {
      subnet_id = yandex_vpc_subnet.public-central-b.id
      nat=true

  }
  metadata = {
    user-data = "${file("/home/ivan/terraform/meta-kibana.txt")}"
  }

}
```
Созданные ВМ

![VM.jpg](https://github.com/Skiledqo/sys-diplom/blob/main/pictures/VM.jpg)

Созданная VPC

![VPC.jpg](https://github.com/Skiledqo/sys-diplom/blob/main/pictures/VPC.jpg)

## Сайт

На созданных ВМ (Web-Server1 и Web-Server2) установлен сервер Nginx.
Также настроен балансировщик нагрузки для этих серверов.

![Balancer.jpg](https://github.com/Skiledqo/sys-diplom/blob/main/pictures/Balancer.jpg)

Пример, как при обращении к веб-серверу меняется источник ответа

![Web-server1.jpg](https://github.com/Skiledqo/sys-diplom/blob/main/pictures/Web-server1.jpg)

![Web-server2.jpg](https://github.com/Skiledqo/sys-diplom/blob/main/pictures/Web-server2.jpg)

Протестируйте сайт curl -v <публичный IP балансера>:80

![Curl.jpg](https://github.com/Skiledqo/sys-diplom/blob/main/pictures/Curl.jpg)

## Мониторинг 

Для развертывания системы мониторинга Zabbix на все ВМ был использован Ansible, установленный на Bastion Host

Конфигурация playbook для Zabbix-Agent и Zabbix-Server

``` bash
---
- name: Install Zabbix Agent
  hosts: zabbix_agents
  tasks:
    - name: Install Zabbix Agent
      apt:
        name: zabbix-agent
        state: latest  
      become: yes
      tags:
        - zabbix-agent

- name: Install Zabbix Server
  hosts: zabbix_server
  tasks:
    - name: Install PostgreSQL and required packages
      apt:
        name: 
          - zabbix-server-pgsql  # Пакет Zabbix Server
          - postgresql            # Пакет PostgreSQL
          - postgresql-contrib    # Пакет дополнений PostgreSQL
          - postgresql-libs       # Пакет библиотек PostgreSQL
          - zabbix-pgsql          # Драйвер PostgreSQL для Zabbix
        state: latest
      become: yes
      tags:
        - zabbix-server
```
Хосты Zabbix

![Hosts.jpg](https://github.com/Skiledqo/sys-diplom/blob/main/pictures/Hosts.jpg)

Дашборд 

![Dashboard.jpg](https://github.com/Skiledqo/sys-diplom/blob/main/pictures/Dashboard.jpg)























