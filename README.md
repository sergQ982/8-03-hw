# 10.4 Отказоустойчивость в облаке

## Задание 1 

Возьмите за основу [решение к заданию 1 из занятия «Подъём инфраструктуры в Яндекс Облаке»](https://github.com/netology-code/sdvps-homeworks/blob/main/7-03.md#задание-1).

1. Теперь вместо одной виртуальной машины сделайте terraform playbook, который:

- создаст 2 идентичные виртуальные машины. Используйте аргумент [count](https://www.terraform.io/docs/language/meta-arguments/count.html) для создания таких ресурсов;
- создаст [таргет-группу](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/lb_target_group). Поместите в неё созданные на шаге 1 виртуальные машины;
- создаст [сетевой балансировщик нагрузки](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/lb_network_load_balancer), который слушает на порту 80, отправляет трафик на порт 80 виртуальных машин и http healthcheck на порт 80 виртуальных машин.

Рекомендуем изучить [документацию сетевого балансировщика нагрузки](https://cloud.yandex.ru/docs/network-load-balancer/quickstart) для того, чтобы было понятно, что вы сделали.

2. Установите на созданные виртуальные машины пакет Nginx любым удобным способом и запустите Nginx веб-сервер на порту 80.

3. Перейдите в веб-консоль Yandex Cloud и убедитесь, что: 

- созданный балансировщик находится в статусе Active,
- обе виртуальные машины в целевой группе находятся в состоянии healthy.

4. Сделайте запрос на 80 порт на внешний IP-адрес балансировщика и убедитесь, что вы получаете ответ в виде дефолтной страницы Nginx.

*В качестве результата пришлите:*

### *1. Terraform Playbook.*
#### Terraform: 
```bash
data "yandex_compute_image" "debian" {
  family = "debian-11"
}

resource "yandex_vpc_network" "web" {
  name = "${var.project_name}-network"
}

resource "yandex_vpc_subnet" "web" {
  name           = "${var.project_name}-subnet"
  zone           = var.zone
  network_id     = yandex_vpc_network.web.id
  v4_cidr_blocks = ["10.10.0.0/24"]
}

resource "yandex_vpc_security_group" "web" {
  name        = "${var.project_name}-sg"
  network_id  = yandex_vpc_network.web.id
  description = "Allow SSH, HTTP and outbound traffic for homework web VMs"

  ingress {
    protocol       = "TCP"
    description    = "HTTP from internet"
    v4_cidr_blocks = ["0.0.0.0/0"]
    port           = 80
  }

  ingress {
    protocol          = "TCP"
    description       = "Health checks from network load balancer"
    predefined_target = "loadbalancer_healthchecks"
    port              = 80
  }

  ingress {
    protocol       = "TCP"
    description    = "SSH administration"
    v4_cidr_blocks = ["0.0.0.0/0"]
    port           = 22
  }
}

resource "yandex_compute_instance" "web" {
  count = 2

  name        = "${var.project_name}-web-${count.index + 1}"
  platform_id = "standard-v3"
  zone        = var.zone

  resources {
    cores  = var.vm_cores
    memory = var.vm_memory
  }

  boot_disk {
    initialize_params {
      image_id = data.yandex_compute_image.debian.id
      size     = 10
      type     = "network-ssd"
    }
  }

  network_interface {
    subnet_id          = yandex_vpc_subnet.web.id
    nat                = true
    security_group_ids = [yandex_vpc_security_group.web.id]
  }

  metadata = {
    ssh-keys  = "debian:${trimspace(file(pathexpand(var.ssh_public_key_path)))}"
    user-data = file("${path.module}/cloud-init.yaml")
  }

  depends_on = [yandex_vpc_subnet.web]
}

resource "yandex_lb_target_group" "web" {
  name = "${var.project_name}-target-group"

  dynamic "target" {
    for_each = yandex_compute_instance.web
    content {
      subnet_id = yandex_vpc_subnet.web.id
      address   = target.value.network_interface[0].ip_address
    }
  }
}

resource "yandex_lb_network_load_balancer" "web" {
  name = "${var.project_name}-nlb"
  type = "external"

  listener {
    name        = "http-listener"
    port        = 80
    target_port = 80
    protocol    = "tcp"

    external_address_spec {
      ip_version = "ipv4"
    }
  }

  attached_target_group {
    target_group_id = yandex_lb_target_group.web.id

    healthcheck {
      name                = "nginx-http-healthcheck"
      interval            = 5
      timeout             = 2
      healthy_threshold   = 2
      unhealthy_threshold = 3

      http_options {
        port = 80
        path = "/"
      }
    }
  }
}

```
### *2. Скриншот статуса балансировщика и целевой группы.*
<img width="1280" height="890" alt="image" src="https://github.com/user-attachments/assets/579c3de0-a712-47a5-9c7c-0f12d8ab347a" />


### *3. Скриншот страницы, которая открылась при запросе IP-адреса балансировщика.*
<img width="1280" height="309" alt="image" src="https://github.com/user-attachments/assets/ce05459d-c76f-46bf-826a-8d613c61bc00" />
<img width="1280" height="435" alt="image" src="https://github.com/user-attachments/assets/3d092e5d-bfd4-45ee-a73c-bc11fd69614d" />



---
