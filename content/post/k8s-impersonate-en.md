На первый взляд система RBAC в k8s достаточно простая и безопасная, но многие забывают или не знают некоторых осоюенностей
В kubernetes есть аналог sudo, благодаря которому можно обойти ограничения в назначенных ролях и получить доступ туда, куда не предполагалось и прочитать скеретные данные или даже получить полный контроль над кластером. И эта возможность есть у любого юзера с дефолтной кластерролью edit. В этой статье попробуем разобраться что это такое и как защититься от такого легального повышения привилегий.

From first view Kubernetes RBAC is simple and secure. While you don't know some nuances.
Kubernetes has its own sudo. It allows to break restrictions in roles and get access to hidden areas. Anyone with default clusterrole edit can do it. This article helps you understand what is it and how to prevent such violations


## Kubernetes Role Based Access Model

There are several approaches to restrict access to resources:
- ABAC - Attribute Based Access Control
- RBAC - Role Based Access Control
- PBAC - Policy Based Access Control
- DAC - Discretionary Access Control
- MAC - Mandatory Access Control
- MLS/MCS - Multi Level Security /  Multi Category Security

Kubernetes может использовать модели доступа: [Node, ABAC, RBAC, WebHook](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#authorization-modules).
Чаще всего приходится работать с ролевой моделью доступа - RBAC. В ней есть несколько взаимосвязанных сущностей:
- роль, описывающая доступ к ресурсам
- пользователь (это может быть User, Group или ServiceAccount)
- Rolebinding - то, что объединяет роли и пользователей 

На самом деле этих сущностей немного больше. Рассмотрим их детальней.
Про RBAC хорошо написано в [документации k8s](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) и я не буду пытаться написать ещё раз. Просто рассмотрим то, без чего остальная статья не имеет смысла. Если читатель знает концепции RBAC k8s - эту часть можно пропусать и сразу переходить к следующей НАЗВАНИЕ ЧАСТИ ТУТ


In Kubernetes you can use bellow access models: [Node, ABAC, RBAC, WebHook](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#authorization-modules).
The most popular model is Role Based Access Control (RBAC) - default system in k8s. It has several related entities:
- role describes access rigths
- user (can be User, Group or ServiceAccount)
- RoleBinding - entity united roles and users

To be honest we have more enitites. Let's look at it deeply.
Kubernetes has amazing [documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) and we won't try to repeat it here. Just will look at the most important things accorded to this article. If you already have known k8s RBAC well just skip this part and go to NEXT_PART_NAME

## Kubernetes RBAC resources
### Role
Роль состоит из списка правил. Каждое правило включает в себя:
- apiGroups - указатель на API groups
- resources - список ресурсов, к которым будет применяться правило. Тут может быть `pods`, `configmaps`, `secrets`, ...
- verbs - доступные операции с перечисленными ресурсами

Role contains rules that represent a set of permissions. Every rule contains:
- apiGroups - pointer to API groups
- resources - list of resources rule will manipulate. Can be `pods`, `configmaps`, `secrets`, ...
- verbs - allowed operations with resources

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

#### Verbs

[Request Verbs](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#determine-the-request-verb)

Verbs описывают что можно делать с ресурсом - как `rwx` в `chmod`. В роли могут использоваться следующие verbs:
- `create` - создать ресурс. HTTP метод `POST`
- get, list, watch - прочитать ресурс. HTTP метод `GET`, `HEAD`
- update - редактировать ресурс. HTTP метод `PUT`
- patch - редактировать ресурс. HTTP метод `PATCH`
- delete, deletecollection - удалить ресурс. HTTP метод `DELETE`

Описанные выше действия очевидны и не требуют пояснений. Самое интересное начинается со следующими verbs:
- [escalate](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#escalate-verb) - создавать и редактировать роли
- [bind](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#bind-verb) - создавать и редактировать биндинги
- [impersonate](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#impersonate-verb) - представиться k8s API другим пользователем, группой или предоставить другие данные (extra)


Verbs describe what can be done with a resource. Like `rwx` in `chmod`. The most popular verbs:
- `create` - create resource. HTTP method POST
- `get`, `list`, `watch` - прочитать ресурс. HTTP метод `GET`, `HEAD`
- `update` - edit resource. HTTP method `PUT`
- `patch` - edit resource. HTTP method `PATCH`
- `delete`, `deletecollection` - delete resource. HTTP method `DELETE`

These actions are obvious and require no explanation. The most interesting verbs are:
- [escalate](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#escalate-verb) - create and edit roles
- [bind](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#bind-verb) - create and edit rolebindings and clusterrolebindings
- [impersonate](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#impersonate-verb) - This verb allows users to impersonate and gain the rights of other users in the cluster, other group or get other data (extra)


### ServiceAccounts, Users, Groups

[ServiceAccount](https://kubernetes.io/docs/concepts/security/service-accounts/)(дальше - SA) - тип учетной записи в k8s, используемый подами, системными компонентами и всем, что не кожаный мешок. В качестве аутентификатора SA использует [токен JWT](https://www.rfc-editor.org/rfc/rfc7519.html). Посмотрим подробнее на создание SA и его JWT токен.
Создание SA:
`k create sa mysa`
`k get sa mysa -oyaml`
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mysa
  namespace: defaul
```
Как видим, в манифесте SA нет ничего особенного. Но есть [нюансы](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#opt-out-of-api-credential-automounting) =). Обратите внимаение, что SA - namespaced resource, то есть у SA всегда есть Namespace.


[ServiceAccount](https://kubernetes.io/docs/concepts/security/service-accounts/)(bellow - SA) - a type of non-human account. Application Pods, system components, and entities inside and outside the cluster can use a specific ServiceAccount's credentials to identify as that ServiceAccount. As authenticator uses [JSON Web Token](https://www.rfc-editor.org/rfc/rfc7519.html). Let's look deeper at SA and its JWT.
Create namespace:
```
kubectl create ns rbac
namespace/rbac created
```

Create ServiceAccount:
```
kubectl -n rbac create sa privesc
serviceaccount/privesc created
```
ServiceAccount manifest:
`k get sa -n rbac privesc -oyaml`
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: privesc
  namespace: rbac
```

#### JWT
Сгенерируем токен для аутентификации от имени SA:
`TOKEN=$(k create token mysa)`
СГЕНЕРИТЬ И ПОКАЗАТЬ
Получили JWT. 
> Обычно по-русски пишут JWT токен, что является [плеоназмом](https://ru.wikipedia.org/wiki/%D0%90%D0%B1%D0%B1%D1%80%D0%B5%D0%B2%D0%B8%D0%B0%D1%82%D1%83%D1%80%D0%B0#%D0%A2%D0%B0%D0%B2%D1%82%D0%BE%D0%BB%D0%BE%D0%B3%D0%B8%D1%87%D0%B5%D1%81%D0%BA%D0%BE%D0%B5_%D1%81%D0%BE%D0%BA%D1%80%D0%B0%D1%89%D0%B5%D0%BD%D0%B8%D0%B5): JSON Web Token токен. Поэтому я буду писать просто JWT

JWT состоит из трех частей, разделенных точками. Первые две части - закодированный в base64 текст, последняя - подпись.
`echo $TOKEN | jq -R ' split(".") | select(length > 0) | .[0],.[1] | @base64d | fromjson'`
ПОКАЗАТЬ ЧТО ПОЛУЧИЛОСЬ


Let's generate token to authenticate as SA:
`TOKEN=$(k -n rbac create token privesc --duration=8h)`
```
echo $TOKEN
eyJhbGciOiJSUzI1NiIsImtpZCI6ImxrNzcybkhfVXZiZW1YSXV0S1BaZDlxNUlFOTRjX1Y1M1o3RWhvLWRsbm8ifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzEyMzI0NjYxLCJpYXQiOjE3MTIyOTU4NjEsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJyYmFjIiwic2VydmljZWFjY291bnQiOnsibmFtZSI6InByaXZlc2MiLCJ1aWQiOiJkODc1NmY0NS01ZDJjLTQ0YjQtYWFjOS02NDU1MjcwNDViZTMifX0sIm5iZiI6MTcxMjI5NTg2MSwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OnJiYWM6cHJpdmVzYyJ9.GxJKpZevOkFwksFsA8ZPU5qLLwQdl6D3Rlt1gU-2feExcy6GadGQJlumrrpq-ih0Ufgm7YUz4jRsNld9yXT93nu27sPyxkMSjMT4rAdfFAV59Q8Z6kFyzOjuJBsEEzErB2Oft5KcGVSXBh01KWvHU8vPvBHaS_JgSV0yym3-9ruGh4eARwc3lbPZi9_PF-P8x0gCvpqaEZWF_aDjxAlcCxlkZjC2ADOHtiVlnBrDt1fqheOZ-W2BKxQ8-z9OG7PMo_x6G6VM2EQIGmY3tzyWd1gMB6bDRrWfSWjj0EPzqdXGov6w-znmzobWHJQN4BoeXBBDJA7BGUIA8VphXHu7yw
```

Now we have JWT. It contains 3 parts divided by dots. First and second parts - base64 encoded text, last part - signature. It's possible to base64 decode first 2 parts and read data:
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


#### Users, Groups

Это может показаться странным, но API k8s не имеет понятия юзера или группы. Невозможно создать пользователя или группу. Но эти данные необходимы для авторизации. API Server распознает пользователя по полю CN в Subject сертификата.
ПОКАЗАТЬТ КАК openssl x509 -noout -text -in ~/gcore/tmp/c.crt  

В качестве системы аутентификации k8s может использовать сторонние Identity Providers и интегрироваться с ними по OpenID Connect. Например, keycloak (ССЫЛКА ТУТ)


It looks strange but k8s API doesn't have user and group entities. It's impossible to create user or group via kubectl. But those data are required to authorization. Kubernetes API Server gets user from CN in Subject field of Certificate. Let's look at it:
```
yq '.users[0].user.client-certificate-data | @base64d' ~/.kube/config | openssl x509 -text -noout
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
Field `Subject` contains data about user and access rights:`O = system:masters, CN = kubernetes-admin`.
Kubernetes can use third-party Identity Providers as authentication system and integrate with it by OpenID Connect. For example keycloak, dex (LINK HERE)

### RoleBinding
Ресурс, связывающий роль и пользователя/группу/сервисаккаунт. Содержит всего два поля:
- список Subjects, в котором перечислены субъекты доступа: пользователи и/или группы и/или сервисаккаунты
- roleRef - роль, которая привязывается к  перечисленным субъектам

Resource united roles and users/groupd/serviveAccounts. Has two fields:
- list of Subjects, holds access subjects: user and/or group and/or ServiceAccount
- roleRef - role united with these subjects

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
roleRef:
  # "roleRef" specifies the binding to a Role / ClusterRole
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```

 ### ClusterRole, ClusterRoleBinding
 Ресурсы кластера делятся на namespaced и non-namespaced. Это значит, что первые всегда имеют namespace, а вторые нет (например `node`).

- Role - namespaced ресурс. То есть она описывает доступы только внутри namespace
- ClusterRole - non-namespaced. Описывает права доступа на ресурсы, которые не имеют namespace. Но также может описывать и namespaced. За подробностями - в [документацию](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole)

RoleBinding отличается от ClusterRoleBinding примерно тем же. Подробности опять же в документации

There are namespaced and non-namespaced cluster resources. It means the first always has namespace, and second never (node, for example)

- Role - namespaced resource. It means it manages access rights in particular namespace
- ClusterRole - non-namespaced resource. It manages access right for any resource: namespaced and non-namespaced. Detailed in [documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole)

## Unobvious Kubernetes RBAC nuances
Not very popular but too dangerous nuances in k8s RBAC


### Escalate
[DOC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#restrictions-on-role-creation-or-update): You can only create/update a _role_ if at least one of the following things is true:
- You already have all the permissions contained in the role, at the same scope as the object being modified (cluster-wide for a ClusterRole, within the same namespace or cluster-wide for a Role).
- You are granted explicit permission to perform the `escalate` verb on the roles or clusterroles resource in the rbac.authorization.k8s.io API group.

[Kubernetes RBAC API doesn't allow to escalate privileges by simple redacting Role or RoleBinding. It works at API level and will work even if RBAC authorizer turned off](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#privilege-escalation-prevention-and-bootstrapping). Исключение из этого правила - наличие права `escalate` у роли.
![escalate](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*CBK_TpCOMNyaWyA32nx-bA.png) НАРИСОВАТЬ!


Create role allowed to read-only access to pods and roles in `rbac` namespace:
```
kubectl -n rbac create role view --verb=list,watch,get --resource=role,pod
role.rbac.authorization.k8s.io/view created 
```

Bind role to ServiceAccount `privesc`:
```
kubectl -n rbac create rolebinding view --role=view --serviceaccount=rbac:privesc
rolebinding.rbac.authorization.k8s.io/view created
```

Check it:
```
kubectl auth can-i get role -n rbac --as=system:serviceaccount:rbac:privesc 
yes

kubectl auth can-i update role -n rbac --as=system:serviceaccount:rbac:privesc 
no
```

Create new role allowed role editing in `rbac` namespace:
```
kubectl -n rbac create role edit --verb=update,patch --resource=role                              
role.rbac.authorization.k8s.io/edit created
```

Bind it to ServiceAccount `privesc`:
```
kubectl -n rbac create rolebinding edit --role=edit --serviceaccount=rbac:privesc
rolebinding.rbac.authorization.k8s.io/edit created
```

Check:
```
kubectl auth can-i update role -n rbac --as=system:serviceaccount:rbac:privesc   
yes

kubectl auth can-i delete role -n rbac --as=system:serviceaccount:rbac:privesc
no
```
Теперь для чистоты эксперимента проверим возможности нашего сервисаккаунта. Для этого используем его токен в конфиге. 

Now for the purity of the experiment let's check the capabilities of our account service. Will use its token for it.
`TOKEN=$(kubectl -n rbac create token privesc --duration=8h)`

Придется удалить из конфига старые параметры аутентификации, т.к. [k8s сначала проверяет сертификат пользователя и не будет проверять токен, если данные о сертификате переданы](https://stackoverflow.com/questions/60083889/kubectl-token-token-doesnt-run-with-the-permissions-of-the-token).

We should remove old authentication parameters from config, becaus [k8s firstly checks user's certificate and won't check token if it has know about certificate already](https://stackoverflow.com/questions/60083889/kubectl-token-token-doesnt-run-with-the-permissions-of-the-token).

```
cp ~/.kube/config ~/.kube/rbac.conf
export KUBECONFIG=~/.kube/rbac.conf
kubectl config delete-user kubernetes-admin
kubectl config set-credentials privesc --token=$TOKEN
kubectl config set-context --current --user=privesc
```

`edit` role shows we can edit roles:
`kubectl -n rbac get role edit -oyaml`
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: edit
  namespace: rbac
rules:
- apiGroups:
  - rbac.authorization.k8s.io
  resources:
  - roles
  verbs:
  - update
  - patch
```

Let's try to add new verb (list), which we have already used in other role (view)
```
kubectl -n rbac edit  role edit 
OK
```
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: edit
  namespace: rbac
rules:
- apiGroups:
  - rbac.authorization.k8s.io
  resources:
  - roles
  verbs:
  - update
  - patch
  - list   # <-- added this string
```

Let's try to add new verb (delete), which we haven't used in other roles:
```
kubectl -n rbac edit  role edit

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: edit
  namespace: rbac
rules:
- apiGroups:
  - rbac.authorization.k8s.io
  resources:
  - roles
  verbs:
  - update
  - patch
  - delete   # <-- added this string

error: roles.rbac.authorization.k8s.io "edit" could not be patched: roles.rbac.authorization.k8s.io "edit" is forbidden: user "system:serviceaccount:rbac:privesc" (groups=["system:serviceaccounts" "system:serviceaccounts:rbac" "system:authenticated"]) is attempting to grant RBAC permissions not currently held:
{APIGroups:["rbac.authorization.k8s.io"], Resources:["roles"], Verbs:["delete"]}
```
Kubernetes не позволяет добавлять себе новых прав, которых ещё нет у пользователя - прав, которые не описаны в других ролях, забинденых к этому пользователю

Kubernetes doesn't allow to add new access rights for user or ServiceAccount if it still hasn't have it. If there aren't such access rights in other roles binded to this user or ServiceAccount.

Let's expand ServiceAccount `privesc` access rights. Will add new role with verb `escalate` using admin config:
```
KUBECONFIG=~/.kube/config kubectl -n rbac create role escalate --verb=escalate --resource=role   
role.rbac.authorization.k8s.io/escalate created

KUBECONFIG=~/.kube/config kubectl -n rbac create rolebinding escalate --role=escalate --serviceaccount=rbac:privesc
rolebinding.rbac.authorization.k8s.io/escalate created
```

Check once again - can we add new verb to role:
```
kubectl -n rbac edit  role edit 
role.rbac.authorization.k8s.io/edit edited
```

Теперь это работает. Пользователь может повышать свои привилегии редактируя существующие роли. То есть verb escalate фактически дает права администратора, т.к. пользователь, обладающий привилегиями escalate может выписать себе любые права на неймспейс или кластер, если это указано в resources.

It works now. User can expand his privileges by editing existing role (or by adding new roles if user has `create` privileges). So, verb `escalate` gives admin privileges in fact. If user has `escalate` privileges, he can expand their privileges to namespace admin or even cluster admin.

### Bind
[DOC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#restrictions-on-role-binding-creation-or-update): To allow a user to create/update _role bindings_:
- Grant them a role that allows them to create/update RoleBinding or ClusterRoleBinding objects, as desired.
- Grant them permissions needed to bind a particular role:
    - implicitly, by giving them the permissions contained in the role.
    - explicitly, by giving them permission to perform the bind verb on the particular Role (or ClusterRole).

`Bind` работает аналогично `escalate`. Если `escalate` разрешает редактировать `Role` или `ClusterRole` для повышения привилегий, то `bind` разрешает редактировать `RoleBinding` или `ClusterRoleBinding`. 

`Bind` works similarly `escalate`. `escalate` allows to edit `Role` or `ClusterRole` for privilege escalation, as well as `bind` allows to edit `RoleBinding` or `ClusterRoleBinding`. 

Let's change kubeconfig to admin: `export KUBECONFIG=~/.kube/config`

Remove old roles and bindings:
```
kubectl -n rbac delete rolebinding view edit escalate
kubectl -n rbac delete role view edit escalate
```

Allow ServiceAccount to view and edit rolebinding and pod in namespace:
```
kubectl -n rbac create role view --verb=list,watch,get --resource=role,rolebinding,pod
kubectl -n rbac create rolebinding view --role=view --serviceaccount=rbac:privesc
kubectl -n rbac create role edit --verb=update,patch,create --resource=rolebinding,pod
kubectl -n rbac create rolebinding edit --role=edit --serviceaccount=rbac:privesc
```

Create separate roles to work with pods, but still don't bind it:
```
kubectl -n rbac create role pod-view-edit --verb=get,list,watch,update,patch --resource=pod
kubectl -n rbac create role delete-pod --verb=delete --resource=pod
```

Let's change kubeconfig to ServiceAccount privesc and try to edit rolebinding:
```
export KUBECONFIG=~/.kube/rbac.conf
kubectl -n rbac create rolebinding pod-view-edit --role=pod-view-edit --serviceaccount=rbac:privesc
rolebinding.rbac.authorization.k8s.io/pod-view-edit created
```
Новая роль успешно забиндилась к существующему сервисаккаунту. Обратите внимание, что роль `pod-view-edit` содержит `verbs` и `resources` уже подключенные сервисаккаунту в rolebinding `view` и `edit`.
Теперь попробуем забиндить роль с новым verb `delete`, которого ещё нет в уже подключенных к сервисаккаунту ролях


New role's binded to ServiceAccount successfully. Pay attention role `pod-view-edit` contains `verbs` abd `resources` already binded to ServiceAccount at rolebinding `view` и `edit`:
```
kubectl -n rbac create rolebinding delete-pod --role=delete-pod --serviceaccount=rbac:privesc
error: failed to create rolebinding: rolebindings.rbac.authorization.k8s.io "delete-pod" is forbidden: user "system:serviceaccount:rbac:privesc" (groups=["system:serviceaccounts" "system:serviceaccounts:rbac" "system:authenticated"]) is attempting to grant RBAC permissions not currently held:
{APIGroups:[""], Resources:["pods"], Verbs:["delete"]}
```

Kubernetes не позволяет нам это сделать, несмотря на то, что у нас есть права на редактирование и создание RoleBindings. Исправить это нам поможет право `bind`. Используя админский конфиг выдадим его сервисаккаунту

Kubernetes doesn't allow to do it even with access rights to edit and create RoleBindings. You could fix it with verb `bind`. Let's do it with admin config:
```
KUBECONFIG=~/.kube/config kubectl -n rbac create role bind --verb=bind --resource=role       
role.rbac.authorization.k8s.io/bind created

KUBECONFIG=~/.kube/config kubectl -n rbac create rolebinding bind --role=bind --serviceaccount=rbac:privesc
rolebinding.rbac.authorization.k8s.io/bind created
``` 

Try again to create RoleBinding with new verb `delete`:
```
kubectl -n rbac create rolebinding delete-pod --role=delete-pod --serviceaccount=rbac:privesc
rolebinding.rbac.authorization.k8s.io/delete-pod created
```

It works now.

### Impersonate
[DOC](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#user-impersonation)
Impersonation requests first authenticate as the requesting user, then switch to the impersonated user info.
- A user makes an API call with their credentials and impersonation headers.
- API server authenticates the user.
- API server ensures the authenticated users have impersonation privileges.
- Request user info is replaced with impersonation values.
- Request is evaluated, authorization acts on impersonated user info.

Impersonate это такой sudo для k8s. С правами impersonate пользователь может представиться другим пользователем и выполнять команды от его имени. kubectl имеет опции `--as`, `--as-group`, `--as-uid`, позволяющие выполнить команду от имени юзера, группы или uid соответственно. То есть, если в одной из ролей пользователь получил impersonate, то его можно считать админом неймспейса. Или кластера в худшем варианте.

Имперсонэйт удобно использовать для проверки корректности RBAC - запускать `kubectl auth can-i --as=$USERNAME -n $MANESPACE get pod`.

Хорошие виндовые админы должны помнить, что даже главные админы должны использовать аккаунты с минимальными правами для ежедневных дел, а важные консоли запускать от имени другого юзера с бОльшим количеством прав. В линуксе тот же принцип обеспечивается с помощью `sudo`. Так и в кубернетес - для защиты от случайного удаления важных данных, лучше не разрешать это действие обычному юзеру, а разрешить только админу, а юзеру дать права на impersonate, чтобы он мог использовать `--as=` когда это действительно нужно.



Impersonate verb is like sudo but for k8s instead of Linux. If user has `imeprsonate` access, he can authenticate as other user and run commands as other user. kubectl has options `--as`, `--as-group`, `--as-uid`, which allowed to run command as other user, other group or other uid respectively. So, we can say if any user got impersonate rights, he would be namespace admin or even cluster admin.

Impersonate is usefull to check RBAC rules - admin should run `kubectl auth can-i --as=$USERNAME -n $MANESPACE $VERB $RESOURCE` and check is authorization works as designed.

```
kubectl auth can-i get pod -n rbac --as=system:serviceaccount:rbac:privesc
yes
```

Or look at all user/ServiceAccounts access rights:
```
kubectl auth can-i --list -n rbac --as=system:serviceaccount:rbac:privesc
Resources                                       Non-Resource URLs                     Resource Names   Verbs
roles.rbac.authorization.k8s.io                 []                                    [edit]           [bind escalate]
selfsubjectaccessreviews.authorization.k8s.io   []                                    []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                                    []               [create]
...
rolebindings.rbac.authorization.k8s.io          []                                    []               [update patch create bind escalate list watch get]
roles.rbac.authorization.k8s.io                 []                                    []               [update patch create bind escalate list watch get]
pods                                            []                                    []               [update patch create delete get list watch]
configmaps                                      []                                    []               [update patch create delete]
secrets                                         []                                    []               [update patch create delete]
```

Good system administrators remember: even if you have domain admin access level you have to use limited account to manage your infrastructure and use privileged account only if it needed for such tasks. It called The principle of least privilege. In cloud era this principle realized by impersonate. To prevent accidentally important resources deletion it is possible to create separate ServiceAccount with verb `delete` and allow users to impersonate only with this ServiceAccount ПОКАЗАТЬ КАК!

There is a project [kubectl-sudo](https://github.com/postfinance/kubectl-sudo) implemented such technique as a kubectl plugin.

![impers](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*8XFhTnrLK8xLRr-eNl1PZw.png) НКАРИСОВАТЬ!

Создадим новый сервисаккаунт в неймспейсе rbac с именем impersonator, роль с правами impersonate и забиндим её на новый сервисаккаунт. Обратите внимание, что в роли мы разрешаем представляться только сервисаккаунтом `privesc` и никаким другим:

Let's create new ServiceAccount `impersonator` in namespace `rbac`, role with `impersonate` verb and RoleBinding. Take a look at `--resource-name` parameter - it is allowed to impersonate only as `privesc` ServiceAccount:
```
KUBECONFIG=~/.kube/config kubectl -n rbac create sa impersonator
serviceaccount/impersonator created

KUBECONFIG=~/.kube/config kubectl -n rbac create role impersonate --resource=serviceaccounts --verb=impersonate --resource-name=privesc
role.rbac.authorization.k8s.io/impersonate created

KUBECONFIG=~/.kube/config kubectl -n rbac create rolebinding impersonator --role=impersonate --serviceaccount=rbac:impersonator
rolebinding.rbac.authorization.k8s.io/impersonator created
```

Let's create new context and check access rights:
```
TOKEN=$(KUBECONFIG=~/.kube/config kubectl -n rbac create token impersonator --duration=8h)
kubectl config set-credentials impersonate --token=$TOKEN   
User "impersonate" set.

kubectl config set-context impersonate@kubernetes  --user=impersonate --cluster=kubernetes       
Context "impersonate@kubernetes" created.

kubectl config use-context impersonate@kubernetes                                      
Switched to context "impersonate@kubernetes".

kubectl auth can-i --list -n rbac
Resources                                       Non-Resource URLs                     Resource Names   Verbs
selfsubjectaccessreviews.authorization.k8s.io   []                                    []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                                    []               [create]

...

serviceaccounts                                 []                                    [privesc]        [impersonate]
```

Ничего лишнего нет - только impersonate, как и указано в роли. Но если воспользоваться правом `impersonate` и представиться как сервисаккаунт `privesc`, то видно, что права совпадают с сервисаккаунтом `privesc`:

There is nothing extra - just impersonate, as specified in the role. But if you use the `impersonate` and impersonate as serviceaccount `privesc`, you can see that the rights are the same as serviceaccount `privesc`:

```
kubectl auth can-i --list -n rbac --as=system:serviceaccount:rbac:privesc
Resources                                       Non-Resource URLs                     Resource Names   Verbs
roles.rbac.authorization.k8s.io                 []                                    [edit]           [bind escalate]
selfsubjectaccessreviews.authorization.k8s.io   []                                    []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                                    []               [create]
pods                                            []                                    []               [get list watch update patch delete create]

...

rolebindings.rbac.authorization.k8s.io          []                                    []               [list watch get update patch create bind escalate]
roles.rbac.authorization.k8s.io                 []                                    []               [list watch get update patch create bind escalate]
configmaps                                      []                                    []               [update patch create delete]
secrets                                         []                                    []               [update patch create delete]
```

То есть с `impersonate` сервисаккаунт получил все свои права + все права сервисаккаунта, которым он может представляться.

That is, with `impersonate` the serviceaccount got all its rights + all the rights of the serviceaccount it can represent itself to.

## Mitigation
Система авторизации k8s очень гибкая и позволяет гранулярно настраивать параметры доступа. Даже в тех случаях, когда пользователю необходимо управлять такими чувствительными для безопасности примитивами как Role/ClusterRole и RoleBinding/ClusterRoleBinding. При этом пользователь не сможет повысить свои привилегии и получить доступ к закрытым данным, если администратор явно не позволит ему повышать привилегии.

Повысить привилегии позволяют verbs `escalate` и `bind`. Первый разрешает добавлять новые записи в Role/ClusterRole, а второй - в RoleBinding/ClusterRoleBinding. Это очень мощные инструменты, неправильное использование которых может нанести значительный урон работе кластера. Следует очень внимательно проверять любое использование этих verbs и всегда убеждаться в том, что соблюдается правило Least Privilegies - пользователь должен иметь минимальный набор привилегий, необходимый для работы.

Для ограничения использования этих или любых других правил в манифестах Role/ClusterRole есть поле `resourceNames`, куда можно (и нужно!) вписать имена ресурсов, которые можно использовать.

The k8s authorisation system is very flexible and allows granular configuration of access parameters. Even in cases where the user needs to manage security sensitive primitives such as Role/ClusterRole and RoleBinding/ClusterRoleBinding. The user will not be able to escalate privileges and access sensitive data unless the administrator explicitly allows the user to escalate privileges.

The verbs `escalate`, `bind` and `impersonate` allow to escalate privileges. The former allows adding new entries to Role/ClusterRole the second to RoleBinding/ClusterRoleBinding and the latter to impersonate as a different user. These are very powerful tools, the misuse of which can cause significant damage to a cluster. You should check any use of these verbs very carefully and always make sure that the Least Privilegies rule is followed - the user must have the minimum set of privileges required to operate.

To restrict the use of these or any other rules, Role/ClusterRole manifests have a `resourceNames` field where you can (and should!) enter the names of resources that can be used.

Example allows create ClusterRoleBinding with `roleRef` named "admin","edit","view":
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: role-grantor
rules:
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterroles"]
  verbs: ["bind"]
  resourceNames: ["admin","edit","view"]
```

То же самое можно сделать с escalate. С `bind` права в роли задает админ, а пользователь только может забиндить эту роль на себя, если это разрешено в `resourceNames`, то с `escalate` пользователь может внутри роли прописать любые параметры и стать админом неймспейса или кластера. То есть `escalate` дает больше возможностей, в то время как `bind` ограничивает пользователя. Имейте это в виду, когда приедтся давать эти права пользователям.

The same can be done with escalate as well as impersonate. With `bind` the rights in the role are set by the admin, and the user can only bind this role to himself if it is allowed in `resourceNames`, while with `escalate` the user can write any parameters inside the role and become the admin of a namespace or cluster. That is, `escalate` gives more options, while `bind` restricts the user. Keep this in mind when you have to give these rights to users.

To prevent unathorized access and RBAC misconfiguration you should periodically check cluster RBAC manifests:
```
kubectl get clusterrole -A -oyaml | yq '.items[] | select (.rules[].verbs[] | contains("esalate" | "bind" | "impersonate"))  | .metadata.name'

kubectl get role -A -oyaml | yq '.items[] | select (.rules[].verbs[] | contains("esalate" | "bind" | "impersonate"))  | .metadata.name'
```

Or use automatic systems which monitor creating or editing roles with suspicious content. Falco, for example. СЮДА МОЖНО ПРИЛЕПИТЬ РЕКЛАМУ НАШЕГО Logs as a Service



Kubernetes предоставляет очень гибкие возможности для управления правами доступа. Вместе с гибкостью приходят и сложность поддержки наряду с угрозами безопасности. Для обеспечения надежной и безопасной работы нужно понимать принципы функционирования системы авторизации и типы вспомогательного инструментария, снижающего риски. И помните, что безопасность всей системы равна безопасности её самого слабого звена.

Kubernetes provides very flexible options for managing access rights. Along with flexibility comes support complexity along with security threats. To ensure safe and secure operation, you need to understand how the authorisation system works and the types of support tools that mitigate risks. And remember, the security of the entire system is equal to the security of its weakest link.