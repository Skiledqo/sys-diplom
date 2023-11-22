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
  cloud_id                = "b1gv6siug2vphth1499i"
  folder_id               = "b1g9rkhgo6efp8u29qa9"
  zone                    = "ru-central1-b"
}

resource "yandex_vpc_network" "netology" {
  name = "netology"
}

resource "yandex_vpc_subnet" "bastion" {
  v4_cidr_blocks = ["172.16.50.0/24"]
  zone           = "ru-central1-b"
  network_id     = "${yandex_vpc_network.netology.id}"
}


resource "yandex_vpc_subnet" "private-central-a" {
  v4_cidr_blocks = ["172.16.27.0/24"]
  zone           = "ru-central1-a"
  network_id     = "${yandex_vpc_network.netology.id}"
  route_table_id = yandex_vpc_route_table.nat-route-table.id
}

resource "yandex_vpc_subnet" "private-central-b" {
  v4_cidr_blocks = ["172.16.17.0/24"]
  zone           = "ru-central1-b"
  network_id     = "${yandex_vpc_network.netology.id}"
  route_table_id = yandex_vpc_route_table.nat-route-table.id
}

resource "yandex_vpc_subnet" "private-central-c" {
  v4_cidr_blocks = ["172.16.37.0/24"]
  zone           = "ru-central1-c"
  network_id     = "${yandex_vpc_network.netology.id}"
}

resource "yandex_vpc_subnet" "public-central-a" {
  v4_cidr_blocks = ["172.16.28.0/24"]
  zone           = "ru-central1-a"
  network_id     = "${yandex_vpc_network.netology.id}"
}

resource "yandex_vpc_subnet" "public-central-b" {
  v4_cidr_blocks = ["172.16.18.0/24"]
  zone           = "ru-central1-b"
  network_id     = "${yandex_vpc_network.netology.id}"
}

resource "yandex_vpc_subnet" "public-central-c" {
  v4_cidr_blocks = ["172.16.38.0/24"]
  zone           = "ru-central1-c"
  network_id     = "${yandex_vpc_network.netology.id}"
}

resource "yandex_vpc_gateway" "nat-gateway" {
  name        = "nat-gateway"  
  shared_egress_gateway {}
}

resource "yandex_vpc_route_table" "nat-route-table" {
  network_id = yandex_vpc_network.netology.id

  static_route {
    destination_prefix = "0.0.0.0/0"
    gateway_id     = yandex_vpc_gateway.nat-gateway.id
  }
}



resource "yandex_vpc_security_group" "private" {
  name       = "Private"
  network_id = "${yandex_vpc_network.netology.id}"

  egress {
    protocol       = "ANY"
    description    = "any"
    v4_cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    protocol       = "TCP"
    description    = "http"
    v4_cidr_blocks = ["0.0.0.0/0"]
    port           = 80
  }

  ingress {
    protocol       = "TCP"
    description    = "https"
    v4_cidr_blocks = ["0.0.0.0/0"]
    port           = 443
  }

  ingress {
    protocol          = "TCP"
    description       = "healthchecks"
    predefined_target = "loadbalancer_healthchecks"
    port              = 30080
  }

  ingress {
    protocol       = "TCP"
    description    = "elasticsearch"
    v4_cidr_blocks = ["0.0.0.0/0"]
    port           = 9200
  }

  ingress {
    protocol       = "ANY"
    description    = "zabbix"
    v4_cidr_blocks = ["172.16.18.0/24"]
    port           = 10050
  }

  ingress {
    protocol       = "TCP"
    description    = "Bastion"
    v4_cidr_blocks = ["172.16.50.0/24"]
    port           = 22
  }

}

resource "yandex_vpc_security_group" "bastion" {
  name       = "bastion"
  network_id = "${yandex_vpc_network.netology.id}"

  egress {
    protocol       = "ANY"
    description    = "any"
    v4_cidr_blocks = ["0.0.0.0/0"]
  }
  

  ingress {
    protocol       = "TCP"
    description    = "ssh"
    v4_cidr_blocks = ["0.0.0.0/0"]
    port           = 22
  }
  
  ingress {
    protocol       = "TCP"
    description    = "ssh"
    v4_cidr_blocks = ["172.16.18.0/24","172.16.28.0/24", "172.16.27.0/24", "172.16.37.0/24", "172.16.38.0/24"]
    port           = 22
  }
  
  ingress {
    protocol       = "TCP"
    description    = "zabbix-agent"
    v4_cidr_blocks = ["0.0.0.0/0"]
    port           = 10050
  }


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
      security_group_ids = [yandex_vpc_security_group.private.id]
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
      security_group_ids = [yandex_vpc_security_group.bastion.id]
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
      security_group_ids = [yandex_vpc_security_group.private.id]
  }
  metadata = {
    user-data = "${file("/home/ivan/terraform/meta-web-server1.txt")}"
  }
  
  

}

output "instance_ip1" {
    value = yandex_compute_instance.web-server1.network_interface.0.ip_address
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
      security_group_ids = [yandex_vpc_security_group.private.id]
  }
  metadata = {
    user-data = "${file("/home/ivan/terraform/meta-web-server2.txt")}"
  }
  
  
}

output "instance_ip2" {
    value = yandex_compute_instance.web-server2.network_interface.0.ip_address
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
      security_group_ids = [yandex_vpc_security_group.private.id]
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
      security_group_ids = [yandex_vpc_security_group.private.id]
  }
  metadata = {
    user-data = "${file("/home/ivan/terraform/meta-kibana.txt")}"
  }

}

resource "yandex_alb_target_group" "web-servers" {
  name      = "web-servers"

  target {
    subnet_id = "${yandex_vpc_subnet.private-central-b.id}"
    ip_address   = "${yandex_compute_instance.web-server1.network_interface.0.ip_address}"
  }

  target {
    subnet_id = "${yandex_vpc_subnet.private-central-a.id}"
    ip_address   = "${yandex_compute_instance.web-server2.network_interface.0.ip_address}"
  }
}

resource "yandex_alb_backend_group" "web-servers" {
  name      = "web-servers"

  http_backend {
    name = "backend"
    weight = 1
    port = 80
    target_group_ids = ["${yandex_alb_target_group.web-servers.id}"]
    
    load_balancing_config {
      panic_threshold = 50
    }    
    healthcheck {
      timeout = "1s"
      interval = "1s"
      healthcheck_port = 80
      http_healthcheck {
        path  = "/"
      }
    }
    
  }
}

resource "yandex_alb_http_router" "web-servers" {
  name   = "web-servers"
}

resource "yandex_alb_virtual_host" "web-servers" {
  name           = "web-servers"
  http_router_id = yandex_alb_http_router.web-servers.id
  route {
    name = "web-servers"
    http_route {
      http_route_action {
        backend_group_id = yandex_alb_backend_group.web-servers.id
      }
    }
  }
}

resource "yandex_alb_load_balancer" "web-servers" {
  name               = "web-servers"
  network_id         = yandex_vpc_network.netology.id
  security_group_ids = [yandex_vpc_security_group.private.id]

  allocation_policy {
    location {
      zone_id   = "ru-central1-a"
      subnet_id = yandex_vpc_subnet.private-central-a.id
    }

    location {
      zone_id   = "ru-central1-b"
      subnet_id = yandex_vpc_subnet.private-central-b.id
    }

     location {
      zone_id   = "ru-central1-c"
      subnet_id = yandex_vpc_subnet.private-central-c.id
    }
  }

  listener {
    name = "web-servers"
    endpoint {
      address {
        external_ipv4_address {
        }
      }
      ports = [ 80 ]
    }
    http {
      handler {
        http_router_id = yandex_alb_http_router.web-servers.id
      }
    }
  }
}

locals {
  disk_ids = [
    yandex_compute_instance.elasticsearch.boot_disk.0.disk_id,
    yandex_compute_instance.web-server1.boot_disk.0.disk_id,
    yandex_compute_instance.web-server2.boot_disk.0.disk_id,
    yandex_compute_instance.zabbix-server.boot_disk.0.disk_id,
    yandex_compute_instance.kibana.boot_disk.0.disk_id,
    yandex_compute_instance.bastion-nat.boot_disk.0.disk_id
  ]
}

resource "yandex_compute_snapshot_schedule" "snapshots" {
  name = "snapshots"

  schedule_policy {
    expression = "0 10 * * *"
  }

  snapshot_count = 7

  snapshot_spec {
    description = "Ежедневный снимок"
    labels = {
      environment = "production"
    }
  }

  disk_ids = local.disk_ids
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

Файл inventory для Ansible

``` bash
[zabbix_agents]
elasticsearch ansible_host=elasticsearch.ru-central1.internal
kibana ansible_host=kibana.ru-central1.internal
web-server1 ansible_host=web-server1.ru-central1.internal
bastion-nat ansible_host=bastion-nat.ru-central1.internal
web-server2 ansible_host=web-server2.ru-central1.internal

[zabbix_server]
zabbix-server ansible_host=zabbix-server.ru-central1.internal
```

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

    - name: Start Zabbix Agent
      service:
        name: zabbix-agent
        state: started
        enabled: true
```

``` bash
- name: Install Zabbix Server
  hosts: zabbix_server
  vars:
    zabbix_db_name: zabbix
    zabbix_db_user: zabbix
    zabbix_db_password: netology
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

     - name: Create PostgreSQL User for Zabbix
      postgresql_user:
        db: "{{ zabbix_db_name }}"
        name: "{{ zabbix_db_user }}"
        password: "{{ zabbix_db_password }}"
        priv: "ALL"
        state: present

    - name: Create PostgreSQL Database for Zabbix
      postgresql_db:
        name: "{{ zabbix_db_name }}"
        owner: "{{ zabbix_db_user }}"
        encoding: UTF8
        lc_collate: 'en_US.UTF-8'
        lc_ctype: 'en_US.UTF-8'
        state: present
```
Хосты Zabbix

![Hosts.jpg](https://github.com/Skiledqo/sys-diplom/blob/main/pictures/Hosts.jpg)

Дашборд 

![Dashboard.jpg](https://github.com/Skiledqo/sys-diplom/blob/main/pictures/Dashboard.jpg)


## Логи

Т.к в данный момент доступ с территории РФ к репозиториям Elastic.co ограничен, было принято решение развернуть ELK в контейнере Docker. Версия Elasticsearch, Kibana и filebeat 7.17.13.

Страница в логами из Web-server1 и Web-server2 Kibana

![Logs.jpg](https://github.com/Skiledqo/sys-diplom/blob/main/pictures/Logs.jpg)

## Сеть 

VPC и Bastion host были созданы ранее, а также ВМ были распределены в свои подсети.

Настроенные группы безопасности для Бастиона и ВМ из приватной сети

![Security-Bastion.jpg](https://github.com/Skiledqo/sys-diplom/blob/main/pictures/Security-Bastion.jpg)

![Seciruty-Private.jpg](https://github.com/Skiledqo/sys-diplom/blob/main/pictures/Security-Private.jpg)

## Резервное копирование

Настроено расписание для снимков жестких дисков

![Snapshots.jpg](https://github.com/Skiledqo/sys-diplom/blob/main/pictures/Snapshots.jpg)























