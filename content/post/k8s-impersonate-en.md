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

Impersonate is usefull to check RBAC rules - admin should run `kubectl auth can-i --as=$USERNAME -n $MANESPACE get pod` and check is authorization works as designed.

Good system administrators remember: even if you have domain admin access level you have to use limited account to manage your infrastructure and use privileged account only if it needed for such tasks. It called The principle of least privilege. In cloud era this principle realized by impersonate. To prevent accidentally important resources deletion it is possible to create separate ServiceAccount with verb `delete` and allow users to impersonate only with this ServicaAccount ПОКАЗАТЬ КАК!

There is a project [kubectl-sudo](https://github.com/postfinance/kubectl-sudo) implemented such technique as a kubectl plugin.

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
#### Threats
Наличие у пользователя неограниченных прав impersonate в неймспейсе или целом кластере может привести к получению полного контроля к неймспейсу/кластеру. И не только для авторизованных пользователей, но и для нелегитимных. Например, злоумыщленник таким образом может повысить свои привилегии из дефолтного сервисаккаунта дл кластер админа.

Таким образом необходимо мониторить Role/ClusterRole на появление в них `impersonate` и изучать откуда эта запись появляется и, возможно, более тонко настраивать все нюансы. Для отслеживания ЧТО ПРИМЕНЯТЬ ДЛЯ АУДИТА?


Impersonate in k8s is like sudo in Linux. So, 
```
    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.
```
It could give users more than they want. Incorrect imeprsonate configuration could allow users admin access to the whole cluster. More dangerous it could be used by unlegitimate users - hacker can escalate privileges from default serviceAccount to admin.

So, it is necessary to monitor Roles/ClusterRoles to `impersonate` verb and know who or what did it. And correct RBAC manifests  if necessary 

ДО СЮДА СДЕЛАЛ. Добавить ниже! 01.04.24
### Escalate
[DOC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#restrictions-on-role-creation-or-update): You can only create/update a _role_ if at least one of the following things is true:
- You already have all the permissions contained in the role, at the same scope as the object being modified (cluster-wide for a ClusterRole, within the same namespace or cluster-wide for a Role).
- You are granted explicit permission to perform the `escalate` verb on the roles or clusterroles resource in the rbac.authorization.k8s.io API group.

[Kubernetes RBAC API doesn't allow to escalate privileges by simple redacting Role or RoleBinding. It works at API level and will work even if RBAC authorizer turned off](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#privilege-escalation-prevention-and-bootstrapping). Исключение из этого правила - наличие права `escalate` у роли.
![escalate](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*CBK_TpCOMNyaWyA32nx-bA.png) НАРИСОВАТЬ!

Create new namespace:
```
kubectl create ns rbac
namespace/rbac created
```

Create ServiceAccount:
```
kubectl -n rbac create sa escalate
serviceaccount/escalate created
```

Create role allowed to read-only access to pods and roles in `rbac` namespace:
```
kubectl -n rbac create role view --verb=list,watch,get --resource=role,pod
role.rbac.authorization.k8s.io/view created 
```

Bind role to ServiceAccount `escalate`:
```
kubectl -n rbac create rolebinding view --role=view --serviceaccount=rbac:escalate
rolebinding.rbac.authorization.k8s.io/view created
```

Check it:
```
kubectl auth can-i get role -n rbac --as=system:serviceaccount:rbac:escalate 
yes

kubectl auth can-i update role -n rbac --as=system:serviceaccount:rbac:escalate 
no
```

Create new role allowed role editing in `rbac` namespace:
```
kubectl -n rbac create role edit --verb=update,patch --resource=role                              
role.rbac.authorization.k8s.io/edit created
```

Bind it to ServiceAccount `escalate`:
```
kubectl -n rbac create rolebinding edit --role=edit --serviceaccount=rbac:escalate
rolebinding.rbac.authorization.k8s.io/edit created
```

Check:
```
kubectl auth can-i update role -n rbac --as=system:serviceaccount:rbac:escalate   
yes

kubectl auth can-i delete role -n rbac --as=system:serviceaccount:rbac:escalate
no
```
Теперь для чистоты эксперимента проверим возможности нашего сервисаккаунта. Для этого используем его токен в конфиге. 

Now for the purity of the experiment let's check the capabilities of our account service. Will use its token for it.
`TOKEN=$(kubectl -n rbac create token escalate --duration=8h)`

Придется удалить из конфига старые параметры аутентификации, т.к. [k8s сначала проверяет сертификат пользователя и не будет проверять токен, если данные о сертификате переданы](https://stackoverflow.com/questions/60083889/kubectl-token-token-doesnt-run-with-the-permissions-of-the-token).

We should remove old authentication parameters from config, becaus [k8s firstly checks user's certificate and won't check token if it has know about certificate already](https://stackoverflow.com/questions/60083889/kubectl-token-token-doesnt-run-with-the-permissions-of-the-token).

```
cp ~/.kube/config ~/.kube/rbac.conf
export KUBECONFIG=~/.kube/rbac.conf
kubectl config delete-user kubernetes-admin
kubectl config set-credentials escalate --token=$TOKEN
kubectl config set-context --current --user=escalate
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

error: roles.rbac.authorization.k8s.io "edit" could not be patched: roles.rbac.authorization.k8s.io "edit" is forbidden: user "system:serviceaccount:rbac:escalate" (groups=["system:serviceaccounts" "system:serviceaccounts:rbac" "system:authenticated"]) is attempting to grant RBAC permissions not currently held:
{APIGroups:["rbac.authorization.k8s.io"], Resources:["roles"], Verbs:["delete"]}
```
Kubernetes не позволяет добавлять себе новых прав, которых ещё нет у пользователя - прав, которые не описаны в других ролях, забинденых к этому пользователю

Kubernetes doesn't allow to add new access rights for user or ServiceAccount if it still hasn't have it. If there aren't such access rights in other roles binded to this user or ServiceAccount.

Let's expand ServiceAccount `escalate` access rights. Will add new role with verb `escalate` using admin config:
```
KUBECONFIG=~/.kube/config kubectl -n rbac create role escalate --verb=escalate --resource=role   
role.rbac.authorization.k8s.io/escalate created

KUBECONFIG=~/.kube/config kubectl -n rbac create rolebinding escalate --role=escalate --serviceaccount=rbac:escalate
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