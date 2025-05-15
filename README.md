# Домашнее задание к занятию «Вычислительные мощности. Балансировщики нагрузки» - Скрыпников Илья
# Задание 1. Yandex Cloud
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
  source  = "/home/dev/clopro-homeworks-15.2/1.jpg"
  acl     = "public-read"
}
```
![Screenshot_1](https://github.com/user-attachments/assets/0a4cf511-e84e-4a9f-8579-8f88f80f3212)

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

![Screenshot_4](https://github.com/user-attachments/assets/f0d3f5c5-5273-435a-8c20-5afef2b5a959)



