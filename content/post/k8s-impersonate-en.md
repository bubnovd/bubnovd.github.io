В kubernetes есть аналог sudo, благодаря которому можно обойти ограничения в назначенных ролях и получить доступ туда, куда не предполагалось. И эта возможность есть у любого юзера с дефолтной кластерролью edit. В этой статье попробуем разобраться что это такое и как защититься от такого легального повышения привилегий.

Kubernetes has its own sudo. It allows to break restrictions in roles and get access to hidden areas. Anyone with default clusterrole edit can do it. This article helps you understand what is it and how to prevent such violations

Зачем используетя? 
- для дебага рбака обычно


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
The most popular model is Role Based Access Control (RBAC). It has several related entities:
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
- [escalate](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#escalate-verb) - создавать и редактировать роли ЧУЖИЕ?? 
- [bind](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#bind-verb) - создавать и редактировать биндинги ЧУЖИЕ?? 
- [impersonate](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#impersonate-verb) - представиться k8s API другим пользователем, группой или предоставить другие данные (extra)


Verbs describe what can be done with a resource. Like `rwx` in `chmod`. The most popular verbs:
- `create` - create resource. HTTP method POST
- `get`, `list`, `watch` - прочитать ресурс. HTTP метод `GET`, `HEAD`
- `update` - edit resource. HTTP method `PUT`
- `patch` - edit resource. HTTP method `PATCH`
- `delete`, `deletecollection` - delete resource. HTTP method `DELETE`

These actions are obvious and require no explanation. The most interesting verbs are:
- [escalate](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#escalate-verb) - create and edit roles ЧУЖИЕ?? 
- [bind](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#bind-verb) - create and edit rolebindings and clusterrolebindings ЧУЖИЕ?? 
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
`TOKEN=$(k create token mysa)`
СГЕНЕРИТЬ И ПОКАЗАТЬ
Now we have JWT. It contains 3 parts divided by dots. First and second parts - base64 encoded text, last part - signature.
`echo $TOKEN | jq -R ' split(".") | select(length > 0) | .[0],.[1] | @base64d | fromjson'`
ПОКАЗАТЬ ЧТО ПОЛУЧИЛОСЬ


#### Users, Groups

Это может показаться странным, но API k8s не имеет понятия юзера или группы. Невозможно создать пользователя или группу. Но эти данные необходимы для авторизации. API Server распознает пользователя по полю CN в Subject сертификата.
ПОКАЗАТЬТ КАК openssl x509 -noout -text -in ~/gcore/tmp/c.crt  

В качестве системы аутентификации k8s может использовать сторонние Identity Providers и интегрироваться с ними по OpenID Connect. Например, keycloak (ССЫЛКА ТУТ)


It looks strange but k8s API doesn't have user and group entities. It's impossible to create user or group via kubectl. But those data are required to authorization. Kubernetes API Server gets user from CN in Subject field of Certificate.
ПОКАЗАТЬТ КАК openssl x509 -noout -text -in ~/gcore/tmp/c.crt

Kubernetes can use third-party Identity Providers as authentication system and integrate with it by OpenID Connect. For example keycloak, dex

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

ДО СЮДА ВСЕ ХОРОШО!! 4.02.24
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

![impers](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*8XFhTnrLK8xLRr-eNl1PZw.png) НКАРИСОВАТЬ!

https://github.com/postfinance/kubectl-sudo
проверить 
```yaml
# Can impersonate the user "jane.doe@example.com"
- apiGroups: [""]
  resources: ["users"]
  verbs: ["impersonate"]
  resourceNames: ["jane.doe@example.com"]
```
##### Угрозы
https://infosecwriteups.com/the-bind-escalate-and-impersonate-verbs-in-the-kubernetes-cluster-e9635b4fbfc6

- давать юзерам ограниченный конфиг и конфиг админа толкьо для реально админских дел


### Escalate
[DOC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#restrictions-on-role-creation-or-update): You can only create/update a _role_ if at least one of the following things is true:
- You already have all the permissions contained in the role, at the same scope as the object being modified (cluster-wide for a ClusterRole, within the same namespace or cluster-wide for a Role).
- You are granted explicit permission to perform the `escalate` verb on the roles or clusterroles resource in the rbac.authorization.k8s.io API group.

[Kubernetes RBAC API не позволяет повысить привелегии доступа путем редактирования Role или RoleBinding. Это происходит на уровне API и будет работать даже если выключен RBAC authorizer](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#privilege-escalation-prevention-and-bootstrapping). Исключение из этого правила - наличие права `escalate` у роли.
![escalate](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*CBK_TpCOMNyaWyA32nx-bA.png) НАРИСОВАТЬ!

### Bind
[DOC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#restrictions-on-role-binding-creation-or-update): To allow a user to create/update _role bindings_:
- Grant them a role that allows them to create/update RoleBinding or ClusterRoleBinding objects, as desired.
- Grant them permissions needed to bind a particular role:
    - implicitly, by giving them the permissions contained in the role.
    - explicitly, by giving them permission to perform the bind verb on the particular Role (or ClusterRole).

https://kubernetes.io/docs/reference/access-authn-authz/rbac/#restrictions-on-role-binding-creation-or-update
> You can only create/update a role binding if you already have all the permissions contained in the referenced role
что это значить? Можно ли эти пермишены получить от других ролей?

Если прав в роли недостаточно, то ты можешь забиндить себя на другую роль.
![pic](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*6rzbOIuEDvpBfUnZZpeGhA.png) НАРИСОВАТЬ!



### Tools


```
k create ns bubnovd
namespace/bubnovd created
```

```
k get sa -n bubnovd
NAME      SECRETS   AGE
default   0         21s
```

берем токен, чтобы ходить в куб от имени СА default
`TOKEN=$(k -n bubnovd  create token default)`

`k config set-credentials default --token=${TOKEN}`

 добавляем контекст 
`k config set-context default@my-cluster --cluster=my-cluster --user=default`

проверяем, что контекст добавлен
```
k config get-contexts
CURRENT   NAME                                          CLUSTER              AUTHINFO                   NAMESPACE
          default@my-cluster                    my-cluster   default                    
*         my-cluster-admin@my-cluster   my-cluster   my-cluster-admin
```

и проверяем, что он работает
`k config use-context default@my-cluster`
```
 k get pod
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:bubnovd:default" cannot list resource "pods" in API group "" in the namespace "default"
```

так и должно быть, ведь прав мы ещё не выдали

будет два СА: default, sasecret. У дефолт толкьо права на лист подов, у сикрет -на лист сикретс
возвразаемся на админа, чтобы создать СА и роли
`k config use-context my-cluster-admin@my-cluster`

cоздаем са
`k -n bubnovd create sa sasecret`
и роли
```
k -n bubnovd create role podread --resource=pods --verb=get --verb=list --verb=watch
role.rbac.authorization.k8s.io/podread created

k -n bubnovd create role secretread --resource=secrets --verb=get --verb=list --verb=watch
role.rbac.authorization.k8s.io/secretread created
```

биндим роли на са
```
k -n bubnovd create rolebinding default:podread --role=podread --serviceaccount=bubnovd:default
rolebinding.rbac.authorization.k8s.io/default:podread created

k -n bubnovd create rolebinding sasecret:secretread --role=secretread --serviceaccount=bubnovd:sasecret
rolebinding.rbac.authorization.k8s.io/sasecret:secretread created
```
создадим под и сикрет, чтобы было что проверять
```
k -n bubnovd run nginx --image=nginx                                                                   
pod/nginx created

k -n bubnovd create secret generic secret --from-literal=key=value
secret/secret created
```

переключаемся на СА дефолт и проверяем, что поды видно, а сикреты нет
```
k config use-context default@my-cluster                 
Switched to context "default@my-cluster".

k get pod -n bubnovd
NAME    READY   STATUS              RESTARTS   AGE
nginx   0/1     ContainerCreating   0          78s

k get secret -n bubnovd
Error from server (Forbidden): secrets is forbidden: User "system:serviceaccount:bubnovd:default" cannot list resource "secrets" in API group "" in the namespace "bubnovd"
```


снова идем в дамина и создаем роль имперсонате и биндинг ее на дефолт акк
```
k config use-context my-cluster-admin@my-cluster
Switched to context "my-cluster-admin@my-cluster".


k -n bubnovd  create role impersonate --resource=serviceaccounts --verb=impersonate
role.rbac.authorization.k8s.io/impersonate created


k -n bubnovd  create rolebinding default:impersonate --role=impersonate --serviceaccount=bubnovd:default
rolebinding.rbac.authorization.k8s.io/default:impersonate created
```

снова переключаемся на СА 
```
k config use-context default@my-cluster                                  
Switched to context "default@my-cluster".
```
и проверяем возможность чтения сикретов
```
k get secret -n bubnovd                              
Error from server (Forbidden): secrets is forbidden: User "system:serviceaccount:bubnovd:default" cannot list resource "secrets" in API group "" in the namespace "bubnovd"
```
нелья
Но если представиться другим СА, то все можно
```
k get secret -n bubnovd --as=system:serviceaccount:bubnovd:sasecret                                               
NAME     TYPE     DATA   AGE
secret   Opaque   1      5m27s
```




k get clusterrole -A -oyaml | yq '.items[] | select (.rules[].verbs[] | contains( "impersonate"))  | .metadata.name'