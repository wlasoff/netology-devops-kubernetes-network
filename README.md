# Домашнее задание к занятию «Как работает сеть в K8s» - Студент группы FOPS-42 Власов Владимир

### Цель задания

Настроить сетевую политику доступа к подам.

### Чеклист готовности к домашнему заданию

1. Кластер K8s с установленным сетевым плагином Calico.

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Документация Calico](https://www.tigera.io/project-calico/).
2. [Network Policy](https://kubernetes.io/docs/concepts/services-networking/network-policies/).
3. [About Network Policy](https://docs.projectcalico.org/about/about-network-policy).

-----

### Задание 1. Создать сетевую политику или несколько политик для обеспечения доступа

1. Создать deployment'ы приложений frontend, backend и cache и соответсвующие сервисы.
2. В качестве образа использовать network-multitool.
3. Разместить поды в namespace App.
4. Создать политики, чтобы обеспечить доступ frontend -> backend -> cache. Другие виды подключений должны быть запрещены.
5. Продемонстрировать, что трафик разрешён и запрещён.

### Правила приёма работы

1. Домашняя работа оформляется в своём Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.

## Выполнение задания 1. 

## Настройка NetworkPolicy

Нужно настроить сетевые политики Kubernetes таким образом, чтобы была разрешена только следующая цепочка взаимодействия приложений:

```text
frontend  --->  backend  --->  cache
```

Приетом другие соединения между приложениями должны быть запрещены.

---

## Окружение

Для выполнения задания использовался существующий Kubernetes-кластер MicroK8s.

| Параметр | Значение |
|---|---|
| Kubernetes | MicroK8s v1.35.6 |
| ОС | Ubuntu 26.04 LTS |
| Node | `k8s` |
| IP node | `192.168.1.56` |
| Container Runtime | containerd 2.1.6 |
| CNI | Calico |
| Namespace | `app` |
| Образ приложений | `wbitt/network-multitool:latest` |

Calico используется как сетевой плагин и обеспечивает применение Kubernetes NetworkPolicy.

---

## Созданные приложения

В namespace `app` созданы три Deployment:

- `frontend`;
- `backend`;
- `cache`.

Для каждого Deployment создан отдельный Service типа `ClusterIP`.

Манифест приложений:

[`application.yaml`](yaml/application.yaml)

После запуска приложения работают в отдельных Pod:

```text
backend    1/1 Running
cache      1/1 Running
frontend   1/1 Running
```

Сервисы приложений:

```text
backend    ClusterIP   10.152.183.61    80/TCP
cache      ClusterIP   10.152.183.17    80/TCP
frontend   ClusterIP   10.152.183.178   80/TCP
```

---

## Проверка сети до применения NetworkPolicy

До создания политик в namespace `app` NetworkPolicy отсутствовали.

Проверка соединений показала, что все направления были доступны:

```text
frontend   -> backend    : ALLOWED
frontend   -> cache      : ALLOWED
backend    -> frontend   : ALLOWED
backend    -> cache      : ALLOWED
cache      -> frontend   : ALLOWED
cache      -> backend    : ALLOWED
```

Таким образом, до применения политик сетевой доступ между приложениями не был ограничен.

---

## Настройка NetworkPolicy

Для реализации требуемой схемы созданы пять политик.

### 1. Default Deny

Файл:

[`network-policy-default-deny.yaml`](yaml/network-policy-default-deny.yaml)

Политика выбирает все Pod namespace `app`:

```yaml
podSelector: {}
policyTypes:
  - Ingress
  - Egress
```

После её применения ingress и egress для Pod запрещены, если доступ явно не разрешён другими NetworkPolicy.

### 2. Разрешение DNS

Файл:

[`network-policy-dns.yaml`](yaml/network-policy-dns.yaml)

После запрета egress приложениям отдельно разрешён доступ к CoreDNS:

```text
app/* ---> kube-system/CoreDNS

UDP/53
TCP/53
```

CoreDNS выбирается по labels:

```text
namespace:
kubernetes.io/metadata.name=kube-system

pod:
k8s-app=kube-dns
```

### 3. Frontend -> Backend

Файл:

[`network-policy-frontend.yaml`](yaml/network-policy-frontend.yaml)

Для `frontend` разрешён исходящий TCP-трафик на порт `80` только к Pod с label:

```text
app=backend
```

```text
frontend ---> backend:80
```

### 4. Backend

Файл:

[`network-policy-backend.yaml`](yaml/network-policy-backend.yaml)

Ingress:

```text
frontend ---> backend:80
```

Egress:

```text
backend ---> cache:80
```

### 5. Cache

Файл:

[`network-policy-cache.yaml`](yaml/network-policy-cache.yaml)

Для `cache` разрешён входящий TCP-трафик на порт `80` только от Pod с label:

```text
app=backend
```

```text
backend ---> cache:80
```

---

## Итоговая схема доступа

```text
                TCP/80                 TCP/80
frontend  ----------------> backend ----------------> cache
```

Дополнительно всем Pod разрешён DNS-трафик к CoreDNS.

Остальные соединения между `frontend`, `backend` и `cache` запрещены.

---

## Созданные NetworkPolicy

```text
NAME                  POD-SELECTOR
allow-dns             <none>
backend-access        app=backend
cache-from-backend    app=cache
default-deny-all      <none>
frontend-to-backend   app=frontend
```
---

## Проверка DNS после применения политик

Frontend разрешает имя backend:

```text
backend.app.svc.cluster.local
Address: 10.152.183.61
```

Backend разрешает имя cache:

```text
cache.app.svc.cluster.local
Address: 10.152.183.17
```
---

## Проверка разрешённого и запрещённого трафика

```text
--- REQUIRED CONNECTIONS ---

frontend   -> backend    : ALLOWED
backend    -> cache      : ALLOWED

--- FORBIDDEN CONNECTIONS ---

frontend   -> cache      : DENIED
backend    -> frontend   : DENIED
cache      -> backend    : DENIED
cache      -> frontend   : DENIED
```

Разрешена только требуемая цепочка:

```text
frontend -> backend -> cache
```

---

## Скриншоты

### Состояние кластера, приложений, сервисов и NetworkPolicy

![Resources](images/task1-resources.jpg)

### Проверка разрешённых и запрещённых соединений

![Network Policy Test](images/task1-network-policy-test.jpg)

---

## Результат

С помощью Calico и Kubernetes NetworkPolicy настроена схема:

```text
frontend -> backend -> cache
```

При этом:

- `frontend -> backend` разрешён;
- `backend -> cache` разрешён;
- `frontend -> cache` запрещён;
- `backend -> frontend` запрещён;
- `cache -> backend` запрещён;
- `cache -> frontend` запрещён;
- DNS-доступ к CoreDNS сохранён.

Требования задания выполнены.








