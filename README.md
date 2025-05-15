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
