10.4 Отказоустойчивость в облаке
Задание 1
Возьмите за основу решение задания 1 из занятия «Создание инфраструктуры в Яндекс Облаке».

Теперь вместо одной виртуальной машины создайте плейбук terraform, который:
создаст 2 идентичные виртуальные машины. Используйте аргумент count для создания таких ресурсов;
создаст таргет-группу. Поместите в нее виртуальные машины, созданные на шаге 1;
создаст сетевой балансировщик нагрузки, который прослушивает порт 80, отправляет трафик на порт 80 виртуальных машин и выполняет http-проверку работоспособности на порту 80 виртуальных машин.
Рекомендуем изучить документацию по сетевому балансировщику нагрузки, чтобы понять, что вы сделали.

Установите на созданные виртуальные машины пакет Nginx любым удобным способом и запустите веб-сервер Nginx на порту 80.

Перейдите в веб-консоль Yandex Cloud и убедитесь, что:

созданный балансировщик находится в статусе Active,
обе виртуальные машины в целевой группе находятся в состоянии healthy.
Отправьте запрос на порт 80 на внешний IP-адрес балансировщика и убедитесь, что получаете ответ в виде страницы Nginx по умолчанию.
В качестве результата пришлите:

1. data "yandex_compute_image" "debian" {
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

2. Скриншот статуса балансировщика и целевой группы.
png

3. Скриншот страницы, которая открылась при запросе IP-адреса балансировщика.
png png