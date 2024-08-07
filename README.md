# Дипломный практикум в Yandex.Cloud
  * [Цели:](#цели)
  * [Этапы выполнения:](#этапы-выполнения)
     * [Создание облачной инфраструктуры](#создание-облачной-инфраструктуры)
     * [Создание Kubernetes кластера](#создание-kubernetes-кластера)
     * [Создание тестового приложения](#создание-тестового-приложения)
     * [Подготовка cистемы мониторинга и деплой приложения](#подготовка-cистемы-мониторинга-и-деплой-приложения)
     * [Установка и настройка CI/CD](#установка-и-настройка-cicd)
  * [Что необходимо для сдачи задания?](#что-необходимо-для-сдачи-задания)
  * [Как правильно задавать вопросы дипломному руководителю?](#как-правильно-задавать-вопросы-дипломному-руководителю)

**Перед началом работы над дипломным заданием изучите [Инструкция по экономии облачных ресурсов](https://github.com/netology-code/devops-materials/blob/master/cloudwork.MD).**

---
## Цели:

1. Подготовить облачную инфраструктуру на базе облачного провайдера Яндекс.Облако.
2. Запустить и сконфигурировать Kubernetes кластер.
3. Установить и настроить систему мониторинга.
4. Настроить и автоматизировать сборку тестового приложения с использованием Docker-контейнеров.
5. Настроить CI для автоматической сборки и тестирования.
6. Настроить CD для автоматического развёртывания приложения.

---
## Этапы выполнения:

### Создание облачной инфраструктуры

Для начала необходимо подготовить облачную инфраструктуру в ЯО при помощи [Terraform](https://www.terraform.io/).

Особенности выполнения:

- Бюджет купона ограничен, что следует иметь в виду при проектировании инфраструктуры и использовании ресурсов;
Для облачного k8s используйте региональный мастер(неотказоустойчивый). Для self-hosted k8s минимизируйте ресурсы ВМ и долю ЦПУ. В обоих вариантах используйте прерываемые ВМ для worker nodes.

Предварительная подготовка к установке и запуску Kubernetes кластера.

1. Создайте сервисный аккаунт, который будет в дальнейшем использоваться Terraform для работы с инфраструктурой с необходимыми и достаточными правами. Не стоит использовать права суперпользователя

### Создал аккаунт diplom с правами для создания аккауна для кластера кубернетес в облаке и возможностью управлять Key Management Service. Так же создал необходимые ключи доступа

![1](https://github.com/joos-net/devops-diplom/blob/main/img/001.png)
![2](https://github.com/joos-net/devops-diplom/blob/main/img/002.png)

2. Подготовьте [backend](https://www.terraform.io/docs/language/settings/backends/index.html) для Terraform:  
   - Рекомендуемый вариант: S3 bucket в созданном ЯО аккаунте(создание бакета через TF)
   - Альтернативный вариант:  [Terraform Cloud](https://app.terraform.io/)

### Создал S3 bucket joos-netology

![3](https://github.com/joos-net/devops-diplom/blob/main/img/3.png)

```tf
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
  required_version = ">= 0.13"

    // сохраняем tfstate в s3 бакете
    backend "s3" {
    endpoint   = "storage.yandexcloud.net"
    bucket     = "joos-netology"
    region     = "ru-central1"
    key        = "diplom-netology.tfstate"

    skip_region_validation      = true
    skip_metadata_api_check     = true
    skip_credentials_validation = true
    force_path_style            = true

  }
}

```
4. Создайте VPC с подсетями в разных зонах доступности.
5. Убедитесь, что теперь вы можете выполнить команды `terraform destroy` и `terraform apply` без дополнительных ручных действий.
6. В случае использования [Terraform Cloud](https://app.terraform.io/) в качестве [backend](https://www.terraform.io/docs/language/settings/backends/index.html) убедитесь, что применение изменений успешно проходит, используя web-интерфейс Terraform cloud.


### Валидация проходит успешно
**https://gitlab.com/netjoos/diplom-infra/-/jobs/7529075077**

![4](https://github.com/joos-net/devops-diplom/blob/main/img/4.png)

### План проходит успешно
**https://gitlab.com/netjoos/diplom-infra/-/jobs/7529075080**

![5](https://github.com/joos-net/devops-diplom/blob/main/img/5.png)

### Применение проходит успешно
**https://gitlab.com/netjoos/diplom-infra/-/jobs/7529075084**

![05](https://github.com/joos-net/devops-diplom/blob/main/img/005.png)


Ожидаемые результаты:

1. Terraform сконфигурирован и создание инфраструктуры посредством Terraform возможно без дополнительных ручных действий.
2. Полученная конфигурация инфраструктуры является предварительной, поэтому в ходе дальнейшего выполнения задания возможны изменения.

---
### Создание Kubernetes кластера

На этом этапе необходимо создать [Kubernetes](https://kubernetes.io/ru/docs/concepts/overview/what-is-kubernetes/) кластер на базе предварительно созданной инфраструктуры.   Требуется обеспечить доступ к ресурсам из Интернета.

Это можно сделать двумя способами:

1. Рекомендуемый вариант: самостоятельная установка Kubernetes кластера.  
   а. При помощи Terraform подготовить как минимум 3 виртуальных машины Compute Cloud для создания Kubernetes-кластера. Тип виртуальной машины следует выбрать самостоятельно с учётом требовании к производительности и стоимости. Если в дальнейшем поймете, что необходимо сменить тип инстанса, используйте Terraform для внесения изменений.  
   б. Подготовить [ansible](https://www.ansible.com/) конфигурации, можно воспользоваться, например [Kubespray](https://kubernetes.io/docs/setup/production-environment/tools/kubespray/)  
   в. Задеплоить Kubernetes на подготовленные ранее инстансы, в случае нехватки каких-либо ресурсов вы всегда можете создать их при помощи Terraform.
2. Альтернативный вариант: воспользуйтесь сервисом [Yandex Managed Service for Kubernetes](https://cloud.yandex.ru/services/managed-kubernetes)  
  а. С помощью terraform resource для [kubernetes](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/kubernetes_cluster) создать **региональный** мастер kubernetes с размещением нод в разных 3 подсетях      
  б. С помощью terraform resource для [kubernetes node group](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/kubernetes_node_group)
  
Ожидаемый результат:

1. Работоспособный Kubernetes кластер.
2. В файле `~/.kube/config` находятся данные для доступа к кластеру.
3. Команда `kubectl get pods --all-namespaces` отрабатывает без ошибок.


### После выполнения шага apply кластер развернут и работоспособен

**https://gitlab.com/netjoos/diplom-infra/-/pipelines/1404622851**

**https://gitlab.com/netjoos/diplom-infra/-/blob/main/k8s.tf?ref_type=heads**

**https://gitlab.com/netjoos/diplom-infra/-/jobs/7529075084**

![6](https://github.com/joos-net/devops-diplom/blob/main/img/06.png)

![7](https://github.com/joos-net/devops-diplom/blob/main/img/7.png)

![8](https://github.com/joos-net/devops-diplom/blob/main/img/8.png)

---
### Создание тестового приложения

Для перехода к следующему этапу необходимо подготовить тестовое приложение, эмулирующее основное приложение разрабатываемое вашей компанией.

Способ подготовки:

1. Рекомендуемый вариант:  
   а. Создайте отдельный git репозиторий с простым nginx конфигом, который будет отдавать статические данные.  
   б. Подготовьте Dockerfile для создания образа приложения.  
2. Альтернативный вариант:  
   а. Используйте любой другой код, главное, чтобы был самостоятельно создан Dockerfile.

Ожидаемый результат:

1. Git репозиторий с тестовым приложением и Dockerfile.
2. Регистри с собранным docker image. В качестве регистри может быть DockerHub или [Yandex Container Registry](https://cloud.yandex.ru/services/container-registry), созданный также с помощью terraform.


### Git репозиторий

**https://gitlab.com/netjoos/diplom-site**


### Dockerfile

```Dockerfile
FROM nginx
RUN rm -rf /usr/share/nginx/html/*
COPY source/ /usr/share/nginx/html/
EXPOSE 80
```

### DockerHub c образами

**https://hub.docker.com/r/jooos/netology-dip/tags**

![08](https://github.com/joos-net/devops-diplom/blob/main/img/08.png)

---
### Подготовка cистемы мониторинга и деплой приложения

Уже должны быть готовы конфигурации для автоматического создания облачной инфраструктуры и поднятия Kubernetes кластера.  
Теперь необходимо подготовить конфигурационные файлы для настройки нашего Kubernetes кластера.

Цель:
1. Задеплоить в кластер [prometheus](https://prometheus.io/), [grafana](https://grafana.com/), [alertmanager](https://github.com/prometheus/alertmanager), [экспортер](https://github.com/prometheus/node_exporter) основных метрик Kubernetes.
2. Задеплоить тестовое приложение, например, [nginx](https://www.nginx.com/) сервер отдающий статическую страницу.

Способ выполнения:
1. Воспользоваться пакетом [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus), который уже включает в себя [Kubernetes оператор](https://operatorhub.io/) для [grafana](https://grafana.com/), [prometheus](https://prometheus.io/), [alertmanager](https://github.com/prometheus/alertmanager) и [node_exporter](https://github.com/prometheus/node_exporter). Альтернативный вариант - использовать набор helm чартов от [bitnami](https://github.com/bitnami/charts/tree/main/bitnami).

2. Если на первом этапе вы не воспользовались [Terraform Cloud](https://app.terraform.io/), то задеплойте и настройте в кластере [atlantis](https://www.runatlantis.io/) для отслеживания изменений инфраструктуры. Альтернативный вариант 3 задания: вместо Terraform Cloud или atlantis настройте на автоматический запуск и применение конфигурации terraform из вашего git-репозитория в выбранной вами CI-CD системе при любом комите в main ветку. Предоставьте скриншоты работы пайплайна из CI/CD системы.

Ожидаемый результат:
1. Git репозиторий с конфигурационными файлами для настройки Kubernetes.
2. Http доступ к web интерфейсу grafana.
3. Дашборды в grafana отображающие состояние Kubernetes кластера.
4. Http доступ к тестовому приложению.


### Git репозиторий для создания инфраструктуры, ее дальнейшей настройкой и удалением.
**https://gitlab.com/netjoos/diplom-infra**

### Настройка инфраструктуры
**https://gitlab.com/netjoos/diplom-infra/-/jobs/7529075087**


### После успешной настройки получаем рабочий сайт и мониторинг

![09](https://github.com/joos-net/devops-diplom/blob/main/img/09.png)

![9](https://github.com/joos-net/devops-diplom/blob/main/img/009.png)

**[https://app.jo-os.ru](https://app.jo-os.ru)**

![10](https://github.com/joos-net/devops-diplom/blob/main/img/010.png)

**[https://monitoring.jo-os.ru](https://monitoring.jo-os.ru)**

![11](https://github.com/joos-net/devops-diplom/blob/main/img/11.png)

![12](https://github.com/joos-net/devops-diplom/blob/main/img/12.png)

---
### Установка и настройка CI/CD

Осталось настроить ci/cd систему для автоматической сборки docker image и деплоя приложения при изменении кода.

Цель:

1. Автоматическая сборка docker образа при коммите в репозиторий с тестовым приложением.
2. Автоматический деплой нового docker образа.

Можно использовать [teamcity](https://www.jetbrains.com/ru-ru/teamcity/), [jenkins](https://www.jenkins.io/), [GitLab CI](https://about.gitlab.com/stages-devops-lifecycle/continuous-integration/) или GitHub Actions.

Ожидаемый результат:

1. Интерфейс ci/cd сервиса доступен по http.
2. При любом коммите в репозиторие с тестовым приложением происходит сборка и отправка в регистр Docker образа.
3. При создании тега (например, v1.0.0) происходит сборка и отправка с соответствующим label в регистри, а также деплой соответствующего Docker образа в кластер Kubernetes.

### Git репозиторий сайта

**https://gitlab.com/netjoos/diplom-site**

### Меняем старую подпись

![13](https://github.com/joos-net/devops-diplom/blob/main/img/013.png)

### Видим успешную сборку и отправку

**https://gitlab.com/netjoos/diplom-site/-/jobs/7529276052**

![14](https://github.com/joos-net/devops-diplom/blob/main/img/014.png)

**https://gitlab.com/netjoos/diplom-site/-/pipelines/1404652095**

![15](https://github.com/joos-net/devops-diplom/blob/main/img/0015.png)

https://hub.docker.com/r/jooos/netology-dip/tags

![16](https://github.com/joos-net/devops-diplom/blob/main/img/016.png)

### Видим изменения на сайте

![17](https://github.com/joos-net/devops-diplom/blob/main/img/017.png)

### Создадим Tag 2.0

![18](https://github.com/joos-net/devops-diplom/blob/main/img/18.png)

### Ждем выполнения работы пайплайна

**https://gitlab.com/netjoos/diplom-site/-/pipelines/1404667136**

![19](https://github.com/joos-net/devops-diplom/blob/main/img/19.png)

**https://hub.docker.com/r/jooos/netology-dip/tags**

![20](https://github.com/joos-net/devops-diplom/blob/main/img/20.png)

![21](https://github.com/joos-net/devops-diplom/blob/main/img/21.png)


---
## Что необходимо для сдачи задания?

1. Репозиторий с конфигурационными файлами Terraform и готовность продемонстрировать создание всех ресурсов с нуля.
2. Пример pull request с комментариями созданными atlantis'ом или снимки экрана из Terraform Cloud или вашего CI-CD-terraform pipeline.
3. Репозиторий с конфигурацией ansible, если был выбран способ создания Kubernetes кластера при помощи ansible.
4. Репозиторий с Dockerfile тестового приложения и ссылка на собранный docker image.
5. Репозиторий с конфигурацией Kubernetes кластера.
6. Ссылка на тестовое приложение и веб интерфейс Grafana с данными доступа.
7. Все репозитории рекомендуется хранить на одном ресурсе (github, gitlab)

