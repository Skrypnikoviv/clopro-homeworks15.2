# Домашнее задание к занятию «Вычислительные мощности. Балансировщики нагрузки» - Скрыпников Илья
### Подготовка к выполнению задания

1. Домашнее задание состоит из обязательной части, которую нужно выполнить на провайдере Yandex Cloud, и дополнительной части в AWS (выполняется по желанию). 
2. Все домашние задания в блоке 15 связаны друг с другом и в конце представляют пример законченной инфраструктуры.  
3. Все задания нужно выполнить с помощью Terraform. Результатом выполненного домашнего задания будет код в репозитории. 
4. Перед началом работы настройте доступ к облачным ресурсам из Terraform, используя материалы прошлых лекций и домашних заданий.

---
## Задание 1. Yandex Cloud 

**Что нужно сделать**

1. Создать бакет Object Storage и разместить в нём файл с картинкой:

 - Создать бакет в Object Storage с произвольным именем (например, _имя_студента_дата_).
 - Положить в бакет файл с картинкой.
 - Сделать файл доступным из интернета.
 
2. Создать группу ВМ в public подсети фиксированного размера с шаблоном LAMP и веб-страницей, содержащей ссылку на картинку из бакета:

 - Создать Instance Group с тремя ВМ и шаблоном LAMP. Для LAMP рекомендуется использовать `image_id = fd827b91d99psvq5fjit`.
 - Для создания стартовой веб-страницы рекомендуется использовать раздел `user_data` в [meta_data](https://cloud.yandex.ru/docs/compute/concepts/vm-metadata).
 - Разместить в стартовой веб-странице шаблонной ВМ ссылку на картинку из бакета.
 - Настроить проверку состояния ВМ.
 
3. Подключить группу к сетевому балансировщику:

 - Создать сетевой балансировщик.
 - Проверить работоспособность, удалив одну или несколько ВМ.
4. (дополнительно)* Создать Application Load Balancer с использованием Instance group и проверкой состояния.

Полезные документы:

- [Compute instance group](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/compute_instance_group).
- [Network Load Balancer](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/lb_network_load_balancer).
- [Группа ВМ с сетевым балансировщиком](https://cloud.yandex.ru/docs/compute/operations/instance-groups/create-with-balancer).


# Решение 1.
## storage.tf
```hcl
resource "yandex_iam_service_account_static_access_key" "sa-static-key" {
  service_account_id = var.service_account_id
  description        = "static access key for object storage"
}

resource "yandex_storage_bucket" "bucket" {
  access_key = yandex_iam_service_account_static_access_key.sa-static-key.access_key
  secret_key = yandex_iam_service_account_static_access_key.sa-static-key.secret_key
  bucket = "${var.student_name}-${var.date}"
  acl    = "public-read"
}

resource "yandex_storage_object" "image" {
  access_key = yandex_iam_service_account_static_access_key.sa-static-key.access_key
  secret_key = yandex_iam_service_account_static_access_key.sa-static-key.secret_key
  bucket  = yandex_storage_bucket.bucket.bucket
  key     = "1.jpg"
  source  = "/home/bev/clopro-homeworks-15.2/1.jpg"
  acl     = "public-read"
}
```
![Screenshot_1](https://github.com/user-attachments/assets/0a4cf511-e84e-4a9f-8579-8f88f80f3212)
# Решение 2. 
## main.tf
```hcl
resource "yandex_vpc_network" "network" {
  name = "lamp-network"
}

resource "yandex_vpc_subnet" "subnet" {
  name           = "lamp-subnet"
  network_id     = yandex_vpc_network.network.id
  zone           = "ru-central1-a"
  v4_cidr_blocks = ["192.168.10.0/24"]
}

resource "yandex_compute_instance_group" "lamp_group" {
  name               = "lamp-instance-group"
  folder_id          = var.folder_id
  service_account_id = var.service_account_id

  instance_template {
    platform_id = "standard-v1"
    resources {
      cores  = 2
      memory = 2
    }

    boot_disk {
      initialize_params {
        image_id = "fd827b91d99psvq5fjit" 
      }
    }

    network_interface {
      subnet_ids = [yandex_vpc_subnet.subnet.id]
      nat        = true
    }

    metadata = {
      user-data = file("${path.module}/cloud-init.yaml")
    }
  }

  scale_policy {
    fixed_scale {
      size = 3
    }
  }

  deploy_policy {
    max_unavailable = 1
    max_creating    = 1
    max_expansion   = 1
    max_deleting    = 1
  }

  allocation_policy {
    zones = ["ru-central1-a"]
  }

  health_check {
    http_options {
      port = 80
      path = "/"
    }
    interval = 5
    timeout = 3
    healthy_threshold = 2
    unhealthy_threshold = 2
  }
}
```
## cloud-init.yaml
```hcl
#cloud-config
write_files:
  - path: /var/www/html/index.html
    content: |
      <html>
      <body>
        <h1>Welcome to LAMP</h1>
        <p>This is a test web page served from a LAMP stack on Yandex Cloud.</p>
        <img src="https://storage.yandexcloud.net/skrypnikov-05-15-2025/1.jpg" alt="Image">
      </body>
      </html>
    owner: www-data:www-data
    permissions: '0644'

runcmd:
  - systemctl restart apache2
  - systemctl enable apache2

users:
  - name: ${var.ssh_user}
    groups: sudo
    shell: /bin/bash
    sudo: 'ALL=(ALL) NOPASSWD:ALL'
    ssh_authorized_keys:
      - ${file("${var.ssh_public_key}")}
```
![Screenshot_2](https://github.com/user-attachments/assets/fea81516-72fb-4473-97db-ee52efedcc63)

![Screenshot_3](https://github.com/user-attachments/assets/6add3269-7d8e-418a-a314-799fdaac129e)

![Screenshot_4](https://github.com/user-attachments/assets/14fee604-368f-4114-8dbf-517afa1e68d7)

# Решение 3. 
## loadbalancer.tf
```hcl
resource "yandex_lb_target_group" "lamp_target_group" {
  name = "lamp-target-group"

  dynamic "target" {
    for_each = yandex_compute_instance_group.lamp_group.instances
    content {
      address   = target.value.network_interface[0].ip_address
      subnet_id = yandex_vpc_subnet.subnet.id
    }
  }
}

resource "yandex_lb_network_load_balancer" "lb" {
  name = "lamp-load-balancer"

  listener {
    name        = "listener-http"
    port        = 80
    target_port = 80
    protocol    = "tcp"
  }

  attached_target_group {
    target_group_id = yandex_lb_target_group.lamp_target_group.id

    healthcheck {
      name = "http-check"
      http_options {
        port = 80
        path = "/"
      }
    }
  }
}
```
![Screenshot_7](https://github.com/user-attachments/assets/969f224e-c7e7-43bb-b82a-f6b994cb7964)
![Screenshot_5](https://github.com/user-attachments/assets/28d49d4e-967b-4cfe-9d89-1a4658833532)
![Screenshot_8](https://github.com/user-attachments/assets/5946af82-004b-4ff4-a30f-68c16ce6ddcd)



