# Домашнее задание к занятию "`Продвинутые методы работы с Terraform`" - `Никулин Михаил Сергеевич`



---

### Задание 1

1. Возьмите из демонстрации к лекции готовый код для создания ВМ с помощью remote-модуля.
2. Создайте одну ВМ, используя этот модуль. В файле cloud-init.yml необходимо использовать переменную для ssh-ключа вместо хардкода. Передайте ssh-ключ в функцию template_file в блоке vars ={} . Воспользуйтесь примером. Обратите внимание, что ssh-authorized-keys принимает в себя список, а не строку.
3. Добавьте в файл cloud-init.yml установку nginx.
4. Предоставьте скриншот подключения к консоли и вывод команды sudo nginx -t.

### Ответ:

Возьмем файл [main.tf](src%2Fmain.tf) из демонстрации, отредактируем его, изменим количество и добавим строчку ```vars = {public_key = var.public_key}``` в функцию ```template_file```:
```
#создаем облачную сеть
resource "yandex_vpc_network" "develop" {
  name = "develop"
}

#создаем подсеть
resource "yandex_vpc_subnet" "develop" {
  name           = "develop-ru-central1-a"
  zone           = "ru-central1-a"
  network_id     = yandex_vpc_network.develop.id
  v4_cidr_blocks = ["10.0.1.0/24"]
}

module "test-vm" {
  source          = "git::https://github.com/udjin10/yandex_compute_instance.git?ref=main"
  env_name        = "develop"
  network_id      = yandex_vpc_network.develop.id
  subnet_zones    = ["ru-central1-a"]
  subnet_ids      = [ yandex_vpc_subnet.develop.id ]
  instance_name   = "web"
  instance_count  = 1
  image_family    = "ubuntu-2004-lts"
  public_ip       = true

  metadata = {
      user-data          = data.template_file.cloudinit.rendered #Для демонстрации №3
      serial-port-enable = 1
  }

}

#Пример передачи cloud-config в ВМ для демонстрации №3
data "template_file" "cloudinit" {
 template = file("./cloud-init.yml")
  vars = {public_key = var.public_key}
}
```
Добавим переменную ```public_key``` в [variables.tf](src%2Fvariables.tf), а в ```personal.auto.tfvars``` вынесем значение ключа:
```
variable "public_key" {
  type    = string
  default = ""
}
```
Откорректируем файл [cloud-init.yml](src%2Fcloud-init.yml) из примера, добавив ссылку на ключ, установку и запуск nginx:
```
#cloud-config
users:
  - name: ubuntu
    groups: sudo
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    ssh_authorized_keys:
      - ${public_key}
package_update: true
package_upgrade: false
packages:
 - vim
 - nginx
runcmd:
  - systemctl enable nginx 
  - systemctl start nginx
```
Создадим ВМ, подключимся по ssh и проверим вывод команды ```sudo nginx -t```:
![task_1_1.png](img%2Ftask_1_1.png)
![task_1_2.png](img%2Ftask_1_2.png)


---

### Задание 2

1. Напишите локальный модуль vpc, который будет создавать 2 ресурса: одну сеть и одну подсеть в зоне, объявленной при вызове модуля, например: ru-central1-a.
2. Вы должны передать в модуль переменные с названием сети, zone и v4_cidr_blocks.
3. Модуль должен возвращать в root module с помощью output информацию о yandex_vpc_subnet. Пришлите скриншот информации из terraform console о своем модуле. Пример: > module.vpc_dev
4. Замените ресурсы yandex_vpc_network и yandex_vpc_subnet созданным модулем. Не забудьте передать необходимые параметры сети из модуля vpc в модуль с виртуальной машиной.
5. Откройте terraform console и предоставьте скриншот содержимого модуля. Пример: > module.vpc_dev.
6. Сгенерируйте документацию к модулю с помощью terraform-docs.

### Ответ:

Создадим директорию [vpc_dev](src%2Fvpc_dev) и разместим в ней файлы 
[main.tf](src%2Fvpc_dev%2Fmain.tf)
```
resource "yandex_vpc_network" "develop" {
  name = var.vpc_name
}

resource "yandex_vpc_subnet" "develop" {
  name = var.vpc_name
  zone = var.default_zone
  network_id     = yandex_vpc_network.develop.id
  v4_cidr_blocks = var.default_cidr
}
```
[outputs.tf](src%2Fvpc_dev%2Foutputs.tf)
```
output "network_id" {
  value = yandex_vpc_network.develop.id
}

output "subnet_id" {
  value = yandex_vpc_subnet.develop.id
}
```
[providers.tf](src%2Fvpc_dev%2Fproviders.tf)
```
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
  required_version = ">=0.13"
}
```
[variables.tf](src%2Fvpc_dev%2Fvariables.tf)
```
variable "default_zone" {
  type = string
  default = "ru-central1-a"
  description = "https://cloud.yandex.ru/docs/overview/concepts/geo-scope"
}

variable "default_cidr" {
  type = list(string)
  default = ["10.0.1.0/24"]
  description = "https://cloud.yandex.ru/docs/vpc/operations/subnet-create"
}

variable "vpc_name" {
  type = string
  default = "develop"
  description = "VPC network&subnet name"
}
```
Внесем правки в первоначальный файл ```main.tf```, ссылка [main_2.tf](src%2Fmain_2.tf), внесем данные по новому модулю:
```
module "vpc_dev" {
  source = "./vpc_dev"
}

module "test-vm" {
  source          = "git::https://github.com/udjin10/yandex_compute_instance.git?ref=main"
  env_name        = "develop"
  network_id      = module.vpc_dev.network_id
  subnet_zones    = ["ru-central1-a"]
  subnet_ids      = [ module.vpc_dev.subnet_id ]
  instance_name   = "web"
  instance_count  = 1
  image_family    = "ubuntu-2004-lts"
  public_ip       = true

  metadata = {
      user-data          = data.template_file.cloudinit.rendered
      serial-port-enable = 1
  }

}

data "template_file" "cloudinit" {
 template = file("./cloud-init.yml")
  vars = {public_key = var.public_key}
}
```
![task_2_1.png](img%2Ftask_2_1.png)

Сгенерируем документацию к модулю с помощью terraform-docs:

<!-- BEGIN_TF_DOCS -->
## Requirements

| Name | Version |
|------|--------|
| <a name="requirement_terraform"></a> [terraform](#requirement\_terraform) | >=0.13 |
| <a name="requirement_yandex"></a> [yandex](#requirement\_yandex) | 0.99.1 |

## Providers

| Name | Version |
|------|---------|
| <a name="provider_yandex"></a> [yandex](#provider\_yandex) | 0.99.1 |

## Modules

No modules.

## Resources

| Name | Type |
|------|------|
| [yandex_vpc_network.develop](https://registry.terraform.io/providers/yandex-cloud/yandex/0.99.1/docs/resources/vpc_network) | resource |
| [yandex_vpc_subnet.develop](https://registry.terraform.io/providers/yandex-cloud/yandex/0.99.1/docs/resources/vpc_subnet) | resource |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_default_cidr"></a> [default\_cidr](#input\_default\_cidr) | https://cloud.yandex.ru/docs/vpc/operations/subnet-create | `list(string)` | <pre>[<br>  "10.0.1.0/24"<br>]</pre> | no |
| <a name="input_default_zone"></a> [default\_zone](#input\_default\_zone) | https://cloud.yandex.ru/docs/overview/concepts/geo-scope | `string` | `"ru-central1-a"` | no |
| <a name="input_vpc_name"></a> [vpc\_name](#input\_vpc\_name) | VPC network&subnet name | `string` | `"develop"` | no |

## Outputs

| Name | Description |
|------|-------------|
| <a name="output_network_id"></a> [network\_id](#output\_network\_id) | n/a |
| <a name="output_subnet_id"></a> [subnet\_id](#output\_subnet\_id) | n/a |
<!-- END_TF_DOCS -->

---

### Задание 3

1. Выведите список ресурсов в стейте.
2. Полностью удалите из стейта модуль vpc.
3. Полностью удалите из стейта модуль vm.
4. Импортируйте всё обратно. Проверьте terraform plan. Изменений быть не должно. Приложите список выполненных команд и скриншоты процессы.

### Ответ:

Произведем необходимые операции, предварительно собрав информацию по ```id```:

![task_3_1.png](img%2Ftask_3_1.png)
![task_3_2.png](img%2Ftask_3_2.png)
![task_3_3.png](img%2Ftask_3_3.png)
![task_3_4.png](img%2Ftask_3_4.png)
![task_3_5.png](img%2Ftask_3_5.png)
![task_3_6.png](img%2Ftask_3_6.png)
![task_3_7.png](img%2Ftask_3_7.png)

После запроса ```terraform plan``` все равно предлагает сделать ВМ прерываемой, хотя она и так такой была, согласно данным. Повторная процедура удаления и импорта в стейт показывает то же самое.
![task_3_8.png](img%2Ftask_3_8.png)

---


