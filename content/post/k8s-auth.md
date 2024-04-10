В ходе подготовки к сертификации Certified Kubernetes Security я разобрался с безопасностью в k8s и решил написать пост о не самых популярных, но очень опасных детялях в RBAC. Пост получился огромный и было решено разбить его на две части с более глубоким погружением в каждую из них, чем было в изначальном варианте. В этом посте рассмотрим систему аутентификации kubernetes, её примитивы, научимся доставать данные из сертификатов и токенов, а так же работать с kubernetes API с помощью curl


## Ролевая модель доступа в Kubernetes

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

Чаще всего приходится работать с ролевой моделью доступа - RBAC, т.к. она используется в k8s по умлочанию. В ней есть несколько взаимосвязанных сущностей, самые важные для нас сегодня:
- роль, описывающая доступ к ресурсам
- пользователь (это может быть User, Group или ServiceAccount)
- Rolebinding - то, что объединяет роли и пользователей 

> ПОПРАВИТЬ ТУТ
Про RBAC хорошо написано в [документации k8s](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) и я не буду пытаться написать ещё раз. Просто рассмотрим то, без чего остальная статья не имеет смысла. Если читатель знает концепции RBAC k8s - эту часть можно пропусать и сразу переходить к следующей НАЗВАНИЕ ЧАСТИ ТУТ

## User VS ServiceAccount
Для аутентификации Kubernetes может использовать две категории пользователей:
- User - используется кожаным мешком для работы с k8s. Для аутентификации использует X.509 сертификат
- ServiceAccount - учетная запись для систем, работающих внутри кластера. Для аутентификации использует JWT токен

> Обычно по-русски пишут JWT токен, что является [плеоназмом](https://ru.wikipedia.org/wiki/%D0%90%D0%B1%D0%B1%D1%80%D0%B5%D0%B2%D0%B8%D0%B0%D1%82%D1%83%D1%80%D0%B0#%D0%A2%D0%B0%D0%B2%D1%82%D0%BE%D0%BB%D0%BE%D0%B3%D0%B8%D1%87%D0%B5%D1%81%D0%BA%D0%BE%D0%B5_%D1%81%D0%BE%D0%BA%D1%80%D0%B0%D1%89%D0%B5%D0%BD%D0%B8%D0%B5): JSON Web Token токен. Поэтому я буду писать просто JWT

## User's Certificate
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

Как видим, `certificate-authority-data` относится ко всему кластеру. Во всех конфигах это поле будет одинаково.


### Посмотрим что содержится внутри сертификатов
Все можно сделать вручную, но я буду применять утилиту `yq` для удобного парсинга `yaml` файлов. Ещё нам понадобится `openssl`. Как правило, он уже есть в большинстве дистрибутивов.

#### CA
```
mkdir ~/certs
yq '.clusters[0].cluster.certificate-authority-data | @base64d' ~/.kube/config  > ~/certs/ca.crt
cat ~/.kube/config
-----BEGIN CERTIFICATE-----
MIIC+jCCAeKgAwIBAgIUPAavs+pcqhHexQ63ql2yGzCDMxIwDQYJKoZIhvcNAQEL
...
yE68Y3UxfwoqnTC5r+vpqyboRreg1H6t5RnKgzO6UfGKs/LZweA4LNz9qG8rhw==
-----END CERTIFICATE-----
```

На первый взгляд - просто набор символов. На второй тоже лучше не станет. Чтобы посмотреть содержимое сертификата используем openssl:
`openssl x509 -text -noout -in ~/certs/ca.crt`
```
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

Рассмотрим некоторые поля в сертификате:
- X509v3 Basic Constraints содержит `CA:TRUE`, что указывает на СA. Этот сертификат может использоваться для подписи и проверки подлинности других сертификатов
- Subject `CN = kubernetes` - "имя пользователя"
- Issuer `CN = kubernetes` - кто выдал сертификат. В СA это поле такое же, как Subject. Потому что сертификат выдан сам себе
- X509v3 Key Usage - как можно использовать сертфиикат
- X509v3 Subject Key Identifier - идентификатор сертфиката. Его мы увидим в сертификатах, подписанных этим СA. ТОЧНО???
- Validity - срок действия сертификата. Если вы теряли кластер потому что проспали обновление сертов - пишите в комменты =)

#### Client Certificate
```
yq '.users[0].user.client-certificate-data | @base64d' ~/.kube/config  > ~/certs/cert.crt
openssl x509 -text -noout -in ~/certs/cert.crt
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