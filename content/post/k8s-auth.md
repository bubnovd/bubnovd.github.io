---
title: "Аутентификация в Kubernetes в деталях. Часть 1: JWT, сертификаты"
date: "2024-04-25T04:02:52Z"
author: bubnovd
authorTwitter: bubnovdnet
image: "/img/k8s-auth/logo.jpg"
description: "Обзор Kuberenetes RBAC и методов аутентификации: сертификаты, JWT"
tags:
- k8s
- authentication
keywords:
- k8s
- kubernetes
- authentication
- certificate
- JWT
- RBAC
- openssl
showFullContent: false
readingTime: true
hideComments: false
---

В ходе подготовки к сертификации Certified Kubernetes Security я разобрался с безопасностью в k8s и решил написать пост о не самых популярных, но очень опасных детялях в RBAC. Пост получился огромный и было решено разбить его на две части с более глубоким погружением в каждую из них, чем было в изначальном варианте. В этом посте рассмотрим систему аутентификации kubernetes, её примитивы, научимся доставать данные из сертификатов и токенов, а так же работать с kubernetes API с помощью curl

# Ролевая модель доступа в Kubernetes

Cуществует несколько подходов к разграничению доступа к ресурсам:
- ABAC - Attribute Based Access Control
- RBAC - Role Based Access Control
- PBAC - Policy Based Access Control
- DAC - Discretionary Access Control
- MAC - Mandatory Access Control
- MLS/MCS - Multi Level Security /  Multi Category Security

Kubernetes может использовать модели доступа: [Node, ABAC, RBAC, WebHook](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#authorization-modules). Модель доступа устанавливается через параметр запуска kube-apiserver, проверить можно так:
```
kubectl  -n kube-system get pod  -l component=kube-apiserver  -oyaml | grep authorization
      - --authorization-mode=Node,RBAC
      - --authorization-mode=Node,RBAC
      - --authorization-mode=Node,RBAC
```

Чаще всего приходится работать с ролевой моделью доступа - RBAC. Она используется в k8s по умлочанию и оперирует несколькими взаимосвязанными сущностями. Самые важные для нас сегодня:
- роль, описывающая доступ к ресурсам
- пользователь (это может быть User, Group или ServiceAccount)
- Rolebinding - то, что объединяет роли и пользователей 

Про RBAC хорошо написано в [документации k8s](https://kubernetes.io/docs/reference/access-authn-authz/rbac/). Самая актуальная информация всегда там. В этой статье рассмотрим взаимосвязь компонентов RBAC и инструменты для дебага.

Для аутентификации Kubernetes может использовать две категории пользователей:
- User - используется кожаным мешком для работы с k8s. Для аутентификации использует X.509 сертификат
- ServiceAccount - учетная запись для систем, работающих внутри кластера. Для аутентификации использует JWT токен

> Обычно по-русски пишут JWT токен, что является [плеоназмом](https://ru.wikipedia.org/wiki/%D0%90%D0%B1%D0%B1%D1%80%D0%B5%D0%B2%D0%B8%D0%B0%D1%82%D1%83%D1%80%D0%B0#%D0%A2%D0%B0%D0%B2%D1%82%D0%BE%D0%BB%D0%BE%D0%B3%D0%B8%D1%87%D0%B5%D1%81%D0%BA%D0%BE%D0%B5_%D1%81%D0%BE%D0%BA%D1%80%D0%B0%D1%89%D0%B5%D0%BD%D0%B8%D0%B5): JSON Web Token токен. Поэтому я буду писать просто JWT

# User's Certificate
Это может показаться странным, но API k8s не имеет понятия юзера или группы. Невозможно создать пользователя или группу внутри кластера. Но эти данные необходимы для авторизации. API Server распознает пользователя по полю CN в Subject сертификата.

Итак, обычные пользователи для аутентификации используют X.509 сертификаты. Эти сертфиикаты можно найти в kubeconfig:
`cat ~/.kube/config`
```yaml
apiVersion: v1
clusters:
-   cluster:
        certificate-authority-data: LS0tLS1CRUdJTicGl1...UlRJRklDQVRFLS0tLS0K
        server: https://10.95.189.14:6443
    name: kubernetes
contexts:
-   context:
        cluster: kubernetes
        user: kubernetes-admin
    name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
-   name: kubernetes-admin
    user:
        client-certificate-data: LS0t...S0tLS0tCg==
        client-key-data: LS0tLS1...ktLS0tLQo=
```
Здесь есть три поля, относящиеся к сертфикатам. Все они закодированы в base64:
- `certificate-authority-data` -  Certificate Authority (CA)
- `client-certificate-data` - сертификат пользователя
- `client-key-data` - ключ пользователя

`certificate-authority-data` относится ко всему кластеру, а не к конкретному юзеру. Во всех конфигах это поле будет одинаково.


## Посмотрим что содержится внутри сертификатов
Все можно сделать вручную, но я буду применять утилиту `yq` для удобного парсинга `yaml` файлов. Ещё нам понадобится `openssl`. Как правило, он уже есть в большинстве дистрибутивов.

## CA
```bash
mkdir ~/certs
yq '.clusters[0].cluster.certificate-authority-data | @base64d' ~/.kube/config  > ~/certs/ca.crt
cat ~/certs/ca.crt
-----BEGIN CERTIFICATE-----
MIIC+jCCAeKgAwIBAgIUPAavs+pcqhHexQ63ql2yGzCDMxIwDQYJKoZIhvcNAQEL
...
yE68Y3UxfwoqnTC5r+vpqyboRreg1H6t5RnKgzO6UfGKs/LZweA4LNz9qG8rhw==
-----END CERTIFICATE-----
```

На первый взгляд - просто набор символов. На второй тоже лучше не станет. Чтобы посмотреть содержимое сертификата используем openssl:
`openssl x509 -text -noout -in ~/certs/ca.crt`
```bash
Certificate:       
    Data:
        Version: 3 (0x2)
        Serial Number:
            3c:06:af:b3:ea:5c:aa:11:de:c5:0e:b7:aa:5d:b2:1b:30:83:33:12
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = kubernetes
        Validity
            Not Before: Mar 22 15:21:44 2024 GMT
            Not After : Mar 20 15:21:44 2034 GMT
        Subject: CN = kubernetes
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:e0:63:6e:11:73:8a:89:3c:de:34:eb:1c:84:43:
                    ...
                    98:27
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Certificate Sign
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Subject Key Identifier: 
                D5:DD:0A:9A:6B:08:9B:94:73:13:11:16:93:7C:E4:C1:24:91:A3:90
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        20:c7:1f:48:d5:36:b6:6e:2e:8d:3e:e2:78:12:8c:1f:1d:97:
        ...
        a8:6f:2b:87
```

> Можно было избежать создания файла и отправить вывод `yq` в stdin `openssl`: `openssl x509 -text -noout -in <(yq '.clusters[0].cluster.certificate-authority-data | @base64d' ~/.kube/config)`

Важные для нас поля в сертификате:
- X509v3 Basic Constraints содержит `CA:TRUE`, что указывает на СA. Этот сертификат может использоваться для подписи и проверки подлинности других сертификатов
- Subject `CN = kubernetes` - "имя пользователя"
- Issuer `CN = kubernetes` - кто выдал сертификат. В СA это поле такое же, как Subject. Потому что сертификат выдан сам себе
- X509v3 Key Usage - как можно использовать сертфиикат
- X509v3 Subject Key Identifier - идентификатор сертфиката. Его мы увидим в сертификатах, подписанных этим СA
- Validity - срок действия сертификата. Если вы теряли кластер потому что проспали обновление сертов - пишите в комменты =)

## Client Certificate
```bash
yq '.users[0].user.client-certificate-data | @base64d' ~/.kube/config  > ~/certs/user.crt
openssl x509 -text -noout -in ~/certs/user.crt
Certificate:                                                                                            
    Data:
        Version: 3 (0x2)
        Serial Number: 2201338778473110666 (0x1e8cb9f4b12e9c8a)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = kubernetes
        Validity
            Not Before: Mar 22 15:21:44 2024 GMT
            Not After : Mar 22 15:26:39 2025 GMT
        Subject: O = system:masters, CN = kubernetes-admin
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:c5:9b:5a:7a:82:cd:1e:c6:8b:d6:66:55:68:2f:
                    ...
                    ba:5b
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage: 
                TLS Web Client Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Authority Key Identifier: 
                D5:DD:0A:9A:6B:08:9B:94:73:13:11:16:93:7C:E4:C1:24:91:A3:90
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        6d:6c:58:62:4a:2a:9e:2a:70:cb:f9:52:64:05:6f:f2:18:72:
        ...
        1f:ad:a3:7a
```

Обратите внимание, что `X509v3 Authority Key Identifier` в сертификате совпадает с `X509v3 Subject Key Identifier` в СA. Это говорит о том, что сертификат подписан нашим СA.

Тут в поле `Subject` есть данные пользователе и его правах: `O = system:masters, CN = kubernetes-admin`:
- `CN = kubernetes-admin` - имя пользователя
- `O = system:masters` - "группа"

> `system:masters` - это супергруппа с неограниченными правами. Она даже не участвует в авторизации - ей разрешено всё. Не думаю, что стоит тут особо выделять ОПАСНОСТЬ использования такого конфига. Последние версии kubeadm создают вместе с ним второй сертификат (конфиг), который "безопасно" (взял в качвычки, потому что нет абсолютной безопасности) использовать - с `Subject: O = kubeadm:cluster-admins, CN = kubernetes-admin`, где `kubeadm:cluster-admins` - обычная группа в RBAC с правами `cluster-admin`

В сертификате для kubelet в Subject будет написано `O = system:nodes, CN = system:node:$NODENAME`, что намекает нам на то, что есть ещё группа `system:nodes`, а сам kubelet ходит в API с именем, содержащим в себе имя ноды. В качестве домашнего задания посмотрите на другие конфиги и сертификаты в `/etc/kubernetes` и `/etc/kubernetes/pki`

> В качестве системы аутентификации k8s может использовать сторонние Identity Providers и интегрироваться с ними по OpenID Connect. Например, [keycloak](https://bubnovd.net/post/keycloak-iac/)

## OpenSSL
Поле key сертификата рассматривать не будем - сейчас там для нас нет ничего интересного. Просто выгрузим его в файл по аналогии с СA и User Certificate:
`yq '.users[0].user.client-key-data | @base64d' ~/.kube/config  > ~/certs/user.key`

Посмотрим как с помощью openssl проверить цепочку сертификатов и ключей. Проверить, что сертификат подписан СA:
```
openssl verify -CAfile ~/certs/ca.crt ~/certs/user.crt
user.crt: OK
```

Проверить, что ключ относится к сертификату:
`diff <(openssl x509 -modulus -noout -in ~/certs/user.crt| openssl md5) <(openssl rsa -modulus -noout -in ~/certs/user.key| openssl md5)`

Тут мы берем modulus от сертификата и ключа, считаем от них md5 хэш и сравниваем вывод - он должен быть одинаков для сертификата и ключа. Можно сравнить и без md5 - результат не изменится. Если `diff` не нашел разницы - значит ключ подходит к сертификату.

## Взаимодействие с kube-apiserver
`kubectl` - всего лишь утилита, которая взаимодействует с Kubernetes посредством REST API. Отправлять запросы к API можно любым другим способом, например, с помощью curl.
Адрес сервера берем из kubeconfig
```bash
SERVER=$(yq '.clusters[0].cluster.server' ~/.kube/config)
```
Попробуем получить список неймспейсов `curl -k $SERVER/api/v1/namespaces`:
```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/api/namespaces\"",
  "reason": "Forbidden",
  "details": {},
  "code": 403
}
```    
Сообщение об ошибке `User "system:anonymous" cannot get path "/api/namespaces"` недвусмысленно говорит о том, что анонимный пользователь не имеет прав на просмотр списка неймспейсов. А наша попытка была без авторизации, поэтому мы anonymous.
Сделаем то же самое, авторизовавшись с помощью наших сертификатов
```bash
curl -k --key ~/certs/user.key --cert ~/certs/user.crt --cacert ~/certs/ca.crt $SERVER/api/v1/namespaces/
```
```json
{
  "kind": "NamespaceList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "5950120"
  },
  "items": [
    {
      "metadata": {
        "name": "calico-apiserver",
        ...
```
На этот раз запрос успешно выполнен. Синтаксис запроса такой `$SERVER/api/$VERSION/namespaces/$NAMESPACE/$KIND`, например, получить список подов из неймспейса default:
```bash
curl -k --key user.key --cert user.crt --cacert ca.crt $SERVER/api/v1/namespaces/default/pods
```
Ответ от сервера я не привожу - он будет в виде JSON, содержащего список подов.

# ServiceAccount's Token
[ServiceAccount](https://kubernetes.io/docs/concepts/security/service-accounts/)(дальше - SA) - тип учетной записи в k8s, используемый подами, системными компонентами и всем, что не кожаный мешок. В качестве аутентификатора SA использует [токен JWT](https://www.rfc-editor.org/rfc/rfc7519.html). Посмотрим подробнее на создание SA и его JWT.
Создаем неймспейс:
```bash
kubectl create ns rbac
namespace/rbac created
```

Создаем сервисаккаунт:
```bash
kubectl -n rbac create sa privesc
serviceaccount/privesc created
```
Манифест сервисаккаунта выглядит так:
`kubectl get sa -n rbac privesc -oyaml`
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: privesc
  namespace: rbac
```
Как видим, в манифесте SA нет ничего особенного. Но как всегда есть [нюансы](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#opt-out-of-api-credential-automounting) =)


Обратите внимание, что у SA всегда есть Namespace, то есть это `namespaced resource`. Ниже мы поговорим об этом подробнее.

## JWT
Сгенерируем токен для аутентификации от имени SA. Используем параметр `duration`, чтобы токен работал продолжительное время. Обычно kubernetes выдает токены на короткий срок, определяемый самим kubernetes.
`TOKEN=$(k -n rbac create token privesc --duration=8h)`
```bash
echo $TOKEN
eyJhbGciOiJSUzI1NiIsImtpZCI6ImxrNzcybkhfVXZiZW1YSXV0S1BaZDlxNUlFOTRjX1Y1M1o3RWhvLWRsbm8ifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzEyMzI0NjYxLCJpYXQiOjE3MTIyOTU4NjEsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJyYmFjIiwic2VydmljZWFjY291bnQiOnsibmFtZSI6InByaXZlc2MiLCJ1aWQiOiJkODc1NmY0NS01ZDJjLTQ0YjQtYWFjOS02NDU1MjcwNDViZTMifX0sIm5iZiI6MTcxMjI5NTg2MSwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OnJiYWM6cHJpdmVzYyJ9.GxJKpZevOkFwksFsA8ZPU5qLLwQdl6D3Rlt1gU-2feExcy6GadGQJlumrrpq-ih0Ufgm7YUz4jRsNld9yXT93nu27sPyxkMSjMT4rAdfFAV59Q8Z6kFyzOjuJBsEEzErB2Oft5KcGVSXBh01KWvHU8vPvBHaS_JgSV0yym3-9ruGh4eARwc3lbPZi9_PF-P8x0gCvpqaEZWF_aDjxAlcCxlkZjC2ADOHtiVlnBrDt1fqheOZ-W2BKxQ8-z9OG7PMo_x6G6VM2EQIGmY3tzyWd1gMB6bDRrWfSWjj0EPzqdXGov6w-znmzobWHJQN4BoeXBBDJA7BGUIA8VphXHu7yw
```
Получили JWT. Он состоит из трех частей, разделенных точками. Первые две части - закодированный в base64 текст, последняя - подпись. Первые две части токена можно декодировать из base64 и получить читаемые даные:
`echo $TOKEN | jq -R ' split(".") | select(length > 0) | .[0],.[1] | @base64d | fromjson'`
```json
{
  "alg": "RS256",
  "kid": "lk772nH_UvbemXIutKPZd9q5IE94c_V53Z7Eho-dlno"
}
{
  "aud": [
    "https://kubernetes.default.svc.cluster.local"
  ],
  "exp": 1712324661,
  "iat": 1712295861,
  "iss": "https://kubernetes.default.svc.cluster.local",
  "kubernetes.io": {
    "namespace": "rbac",
    "serviceaccount": {
      "name": "privesc",
      "uid": "d8756f45-5d2c-44b4-aac9-645527045be3"
    }
  },
  "nbf": 1712295861,
  "sub": "system:serviceaccount:rbac:privesc"
}
```
Содержимое токена:
- `kid` - Key ID
- `sub` - Subject - кому выдан токен
- `iss` - Issuer - кто выдал токен. Наш кластер
- `aud` - Audience - целевая аудитория - на какие ресурсы распространяется действие токена. Опять наш кластер
- `exp` - время действия токена в UnixTime. Перевести в человеческий вид:
```bash
date -j -f %s 1712324661                                                                                  
Fri Apr  5 16:44:21 EEST 2024
```
Подробнее о содержимом токена - в [RFC](https://www.rfc-editor.org/rfc/rfc7519.html).

curl позволяет авторизоваться с помощью токена вместо сертификатов `curl -k -H "Authorization: Bearer $TOKEN" $SERVER/api/v1/namespaces/rbac/pods`.
По умолчанию при создании пода CA и JWT сервисаккаунта монтируются по пути `/var/run/secrets/kubernetes.io/serviceaccount/`. Если злоумышленник попадет внутрь пода (например, через уязвимости в приложении или неправильно настроенный nginx) и прочитает токен, то он сможет обращаться к Kubernetes API и получать или изменять данные в кластере, а может и пойти дальше - повысить привилегии, используя некорректные настройки кластера и стать админом.

Проверим как это работает:
- создадим под `kubectl -n rbac run  nginx --image=nginx`
- запустим шелл внутри пода `kubectl -n rbac exec -it nginx -- /bin/bash`
- получим адрес kube-apiserver
```bash
root@nginx:/var/run/secrets/kubernetes.io/serviceaccount# env | grep KUBERNETES
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT_443_TCP=tcp://192.168.192.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_ADDR=192.168.192.1
KUBERNETES_SERVICE_HOST=192.168.192.1
KUBERNETES_PORT=tcp://192.168.192.1:443
KUBERNETES_PORT_443_TCP_PORT=443
```
- провалимся в директорию с аутентификационными данными внутри пода `cd /var/run/secrets/kubernetes.io/serviceaccount/`, тут есть три файла, объяснять их содержимое не имеет смысла: `ca.crt, namespace, token`. Запишем токен в переменную `TOKEN=$(cat token)`
- с токеном можно обращаться к API kubernetes `curl -k -H "Authorization: Bearer $TOKEN" https://$KUBERNETES_PORT_443_TCP_ADDR/api`
- voila - из пода можно ходить в kube-api. И кто знает что с этим сделает хакер 

По умолчанию поды создаются с дефолтным сервисаккаунтом, а прав у него немного. Но это не повод расслабляться - многие приложения требуют дополнительных прав. Чтобы токен не монтировался в под добавьте в манифест SA опцию `automountServiceAccountToken: false`.

# Role
Роль - всего лишь список правил, которые ещё никому не назначены. Каждое правило включает в себя:
- apiGroups - указатель на API groups
- resources - список ресурсов, к которым будет применяться правило. Тут может быть `pods`, `configmaps`, `secrets`, ...
- verbs - доступные операции с перечисленными ресурсами

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

## Verbs

[Request Verbs](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#determine-the-request-verb)

Verbs описывают что можно делать с ресурсом - как `rwx` в `chmod`. В роли могут использоваться следующие verbs:
- `create` - создать ресурс. HTTP метод `POST`
- `get`, `list`, `watch` - прочитать ресурс. HTTP метод `GET`, `HEAD`
- `update` - редактировать ресурс. HTTP метод `PUT`
- `patch` - редактировать ресурс. HTTP метод `PATCH`
- `delete`, `deletecollection` - удалить ресурс. HTTP метод `DELETE`

# RoleBinding
Ресурс, связывающий роль и пользователя/группу/сервисаккаунт. Содержит всего два поля:
- список Subjects, в котором перечислены субъекты доступа: пользователи и/или группы и/или сервисаккаунты
- roleRef - роль, которая привязывается к  перечисленным субъектам

```yaml
apiVersion: rbac.authorization.k8s.io/v1
# This role binding allows "jane" to read pods in the "default" namespace.
# You need to already have a Role named "pod-reader" in that namespace.
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
# You can specify more than one "subject"
- kind: User
  name: jane # "name" is case sensitive
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: privesc
  apiGroup: rbac.authorization.k8s.io
roleRef:
  # "roleRef" specifies the binding to a Role / ClusterRole
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```

 # ClusterRole, ClusterRoleBinding
 Ресурсы кластера делятся на namespaced и non-namespaced. Это значит, что первые всегда имеют namespace, а вторые нет (например `node`).
 - namespaced resouces: `pod`, `secret`, `ingress`, `configmap`, ...
 - non-namespaced resouces: `namespace`, `node`, `ClusterRole`, `storageclasses`. Больше тут `kubectl api-resources --namespaced=false`

Распределение прав описывают ресурсы Role, RoleBinding, ClusterRole и ClusterRoleBinding

 ![Role VS ClusteRole](/img/k8s-auth/role-clusterrole.png)
- Role - namespaced ресурс. То есть она описывает доступы только внутри неймспейса, в котором эта роль существует. Логично, что если роль работает внутри неймспейса, то и права она может регулировать только для `namespaced` ресурсов
- ClusterRole - non-namespaced. Описывает права доступа на любые ресурсы - namespaced и non-namespaced. Например, дефолтная СlusterRole `view` разрешает читать ресурсы всех типов - `namespaced` и `non-namespaced`

![RoleBinding VS ClusterRoleBinding](/img/k8s-auth/binding-clusterbinding.png)

![rolebinding-schema.png](/img/k8s-auth/rolebinding-schema.png)
- RoleBinding - биндит роли к субъектам доступа (User, Group, ServiceAccount) для описания доступа в одном неймспейсе. RoleBinding может ссылаться на Role и на ClusterRole. При использовании СlusterRole, все разрешения из ClusterRole применятся только к тому неймспейсу, в котором существует этот RoleBinding
![clusterrolebinding-schema.png](/img/k8s-auth/clusterrolebinding-schema.png)
- ClusterRoleBinding биндит только СlusterRole и может давать доступ ко всем ресурсам кластера

Хорошая презентация про RBAC и разницу между Role/ClusterRole и RoleBinding/ClusterRoleBinding [по ссылке](https://www.cncf.io/wp-content/uploads/2020/08/2020_04_Introduction-to-Kubernetes-RBAC.pdf), [видео](https://www.youtube.com/watch?v=B6Ylwugs3t0). Картинки я взял оттуда. Ещё более детально - в [документации](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole)


# Tools
В больших кластерах сложно поддерживать права десятков или сотен пользователей для доступа к разным ресурсам в разных неймспейсах. Вот несколько инструментов, упрощающих жизнь:
- [rbac-manager](https://github.com/FairwindsOps/rbac-manager) - оператор, вводящий новую сущность RBACDefinition. Позволяет биндить роли к неймспейсам по лейблам, а также биндить роли к нескольким субъектам(юзерам) в одном ямле. Очень удобно. Я даже сделал [чарт](https://github.com/bubnovd/rbac-definition-helm) для деплоя ролей, кластерролей и манифестов для rbac-manager из единого места
- [rbac-lookup](https://github.com/FairwindsOps/rbac-lookup) - утилита для анализа прав и юзеров kubernetes
- [audit2rbac](https://github.com/liggitt/audit2rbac) - генерирует RBAC манифесты из аудит логов 

---
Этот пост родился в ходе написания другого поста - про особенности некоторых `verbs` в ролях и их влияние на безопасность. Тот [пост](k8s-auth) получился слишком большим, поэтому я решил опубликовать основы отдельно. Статья про безопасность будет опубликована в ближайшее время, а тут появится ссылка на неё. Так же она будет продублирована в телеграм канале [Mikrotik Ninja](https://t.me/mikrotikninja).

Пока я писал этот пост, в англоязычном интернете появилась [хорошая статья](https://thegreycorner.com/2023/11/15/kubernetes-auth-deep-dive.html) о том же самом. Там есть хорошие примеры про подделку сертификатов и токенов.
