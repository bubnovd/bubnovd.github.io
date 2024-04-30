https://kubernetes.io/docs/reference/access-authn-authz/authentication/#user-impersonation

https://rad.security/blog/what-is-kubernetes-rbac#RBACpolice


- inro
- rbac
- impersonate
- угрозы
- как защититься
- обзор инструментов

Ещё одна угроза - Фggregated ClusterRole 

На первый взляд система RBAC в k8s достаточно простая и безопасная, но многие забывают или не знают некоторых осоюенностей
В kubernetes есть аналог sudo, благодаря которому можно обойти ограничения в назначенных ролях и получить доступ туда, куда не предполагалось и прочитать скеретные данные или даже получить полный контроль над кластером. И эта возможность есть у любого юзера с дефолтной кластерролью edit. В этой статье попробуем разобраться что это такое и как защититься от такого легального повышения привилегий.

## Ролевая модель доступа в Kubernetes


## Ресурсы kubernetes RBAC
![sa-role-rolebinding](/img/k8s-rbac-nuances/sa-role-rolebinding-white.png)
### Role
Роль состоит из списка правил. Каждое правило включает в себя:
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

#### Verbs

[Request Verbs](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#determine-the-request-verb)

Verbs описывают что можно делать с ресурсом - как `rwx` в `chmod`. В роли могут использоваться следующие verbs:
- `create` - создать ресурс. HTTP метод `POST`
- get, list, watch - прочитать ресурс. HTTP метод `GET`, `HEAD`
- update - редактировать ресурс. HTTP метод `PUT`
- patch - редактировать ресурс. HTTP метод `PATCH`
- delete, deletecollection - удалить ресурс. HTTP метод `DELETE`

Описанные выше действия очевидны и не требуют пояснений. Самое интересное начинается со следующими verbs:
- [escalate](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#escalate-verb) - создавать и редактировать роли, в том числе, относящиеся к другим пользователям
- [bind](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#bind-verb) - создавать и редактировать биндинги, в том числе, относящиеся к другим пользователям
- [impersonate](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#impersonate-verb) - представиться k8s API другим пользователем, группой или предоставить другие данные (extra)

### RoleBinding
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

```bash
kubectl create ns rbac
kubectl -n rbac create sa privesc
```
## Неочевидные нюансы k8s RBAC
Наконец-то добрались до того, ради чего писался этот пост.
Не самые популярные, но опасные нюансы в RBAC.

### Escalate
[DOC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#restrictions-on-role-creation-or-update): You can only create/update a _role_ if at least one of the following things is true:
- You already have all the permissions contained in the role, at the same scope as the object being modified (cluster-wide for a ClusterRole, within the same namespace or cluster-wide for a Role).
- You are granted explicit permission to perform the `escalate` verb on the roles or clusterroles resource in the rbac.authorization.k8s.io API group.

[Kubernetes RBAC API не позволяет повысить привелегии путем редактирования Role или RoleBinding. Это происходит на уровне API и будет работать даже если выключен RBAC authorizer](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#privilege-escalation-prevention-and-bootstrapping). Исключение из этого правила - наличие права `escalate` у роли. То есть МОЖНО создавать и редактировать роли, но НЕЛЬЗЯ добавлять юзерам новые права - можно только оперировать лишь теми, которые уже есть у юзера в других ролях
![escalate](/img/k8s-rbac-nuances/escalate-purple-white.png)

Создадим роль `view` с правами только на чтение и роль `edit` с правами на редактирование и проверим может ли пользователь повысить себе привилегии, добавляя новые verbs в роли.

Создаём роль с правами на просмотр и редактирование ролей в неймспейсе rbac и биндим её к SA:
```bash
kubectl -n rbac create role  --verb=list,watch,get --resource=role
role.rbac.authorization.k8s.io/view created

kubectl -n rbac create rolebinding view --role=view --serviceaccount=rbac:privesc
rolebinding.rbac.authorization.k8s.io/view created
```

Проверяем:
```
kubectl auth can-i get role -n rbac --as=system:serviceaccount:rbac:privesc 
yes

kubectl auth can-i update role -n rbac --as=system:serviceaccount:rbac:privesc 
no
```
Сервисаккаунту `privesc` разрешено читать роли, но нельзя их редактировать.

Сделаем роль с правами на чтение и редактирование ролей в неймспейсе rbac:
```
kubectl -n rbac create role edit --verb=update,patch --resource=role                              
role.rbac.authorization.k8s.io/edit created

kubectl -n rbac create rolebinding edit --role=edit --serviceaccount=rbac:privesc
rolebinding.rbac.authorization.k8s.io/edit created
```

Проверяем:
```bash
kubectl auth can-i update role -n rbac --as=system:serviceaccount:rbac:privesc   
yes

kubectl auth can-i delete role -n rbac --as=system:serviceaccount:rbac:privesc
no
```

Редактировать роли разрешено, а удалять запрещено. Всё как написано в ранее созданных ролях.

Теперь для чистоты эксперимента проверим возможности нашего сервисаккаунта, залогинившись в k8s от его имени. Для этого используем его токен в конфиге. 
`TOKEN=$(kubectl -n rbac create token privesc --duration=8h)`

Придется удалить из конфига старые параметры аутентификации, так как [k8s сначала проверяет сертификат пользователя и не будет проверять токен, если данные о сертификате переданы](https://stackoverflow.com/questions/60083889/kubectl-token-token-doesnt-run-with-the-permissions-of-the-token).
```bash
cp ~/.kube/config ~/.kube/rbac.conf
export KUBECONFIG=~/.kube/rbac.conf
kubectl config delete-user kubernetes-admin
kubectl config set-credentials privesc --token=$TOKEN
kubectl config set-context --current --user=privesc
```

В роли edit написано, что мы имеем права на редактирование ролей:
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

Попробуем добавить в роль новый verb `list`, который уже имеется в другой роли `view`
```
kubectl -n rbac edit  role edit 
OK
```
```yaml
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
  - list   # <-- добавлена эта строка
```

Попробуем добавить в роль новый verb `delete`, который не описан в других ролях:
`kubectl -n rbac edit  role edit`
```yaml
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
  - delete   # <-- добавлена эта строка

error: roles.rbac.authorization.k8s.io "edit" could not be patched: roles.rbac.authorization.k8s.io "edit" is forbidden: user "system:serviceaccount:rbac:privesc" (groups=["system:serviceaccounts" "system:serviceaccounts:rbac" "system:authenticated"]) is attempting to grant RBAC permissions not currently held:
{APIGroups:["rbac.authorization.k8s.io"], Resources:["roles"], Verbs:["delete"]}
```
Kubernetes не позволяет добавлять новых прав, которых ещё нет у пользователя - прав, которые не описаны в других ролях, забинденых к этому пользователю. Попробуем пофиксить это.

Как мы убедились - повысить привилегии самому себе нельзя, поэтому дальнейшие действия будем делать от админа - об этом говорит задание переменной `KUBECONFIG=~/.kube/config` перед командой. Расширим права сервисаккаунта privesc - добавим новую роль с verb `escalate` и забиндим её нашему сервисаккаунту:
```bash
KUBECONFIG=~/.kube/config kubectl -n rbac create role escalate --verb=escalate --resource=role   
role.rbac.authorization.k8s.io/escalate created

KUBECONFIG=~/.kube/config kubectl -n rbac create rolebinding escalate --role=escalate --serviceaccount=rbac:privesc
rolebinding.rbac.authorization.k8s.io/escalate created
```

Повторяем то, что не удалось сделать без escalate - добавить delete в существующую роль:
```bash
kubectl -n rbac edit  role edit 
role.rbac.authorization.k8s.io/edit edited
```
```yaml
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
  - delete   # <-- добавлена эта строка
```
Теперь это работает. Пользователь может повышать свои привилегии редактируя существующие роли. То есть verb `escalate` фактически дает права администратора, т.к. пользователь, обладающий привилегиями escalate может выписать себе любые права на неймспейс или кластер, если это указано в resources.

### Bind
[DOC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#restrictions-on-role-binding-creation-or-update): To allow a user to create/update _role bindings_:
- Grant them a role that allows them to create/update RoleBinding or ClusterRoleBinding objects, as desired.
- Grant them permissions needed to bind a particular role:
    - implicitly, by giving them the permissions contained in the role.
    - explicitly, by giving them permission to perform the bind verb on the particular Role (or ClusterRole).

Разрешить пользователю создавать или редактировать rolebindings можно выполнив оба условия:
  - Дать пользователю права создавать/редактировать RoleBinding или ClusterRoleBinding
  - Дать пользователю права биндить роль:
    - явно, описав права в роли
    - неявно, дав ему права `bind` на роль или кластерроль

https://kubernetes.io/docs/reference/access-authn-authz/rbac/#restrictions-on-role-binding-creation-or-update
> You can only create/update a role binding if you already have all the permissions contained in the referenced role
что это значить? Можно ли эти пермишены получить от других ролей?

`Bind` работает аналогично `escalate`. Если `escalate` разрешает редактировать `Role` или `ClusterRole` для повышения привилегий, то `bind` разрешает редактировать `RoleBinding` или `ClusterRoleBinding`. 
Если прав в роли недостаточно, то ты можешь забиндить себя на другую роль.
![pic](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*6rzbOIuEDvpBfUnZZpeGhA.png) НАРИСОВАТЬ!

Переключимся на админский конфиг: `export KUBECONFIG=~/.kube/config`

Удалим созданные ранее роли и биндинги
```
kubectl -n rbac delete rolebinding view edit escalate
kubectl -n rbac delete role view edit escalate
```

Дадим сервисаккаунту права на просмотр и редактирование rolebinding и pod в своём неймспейсе:
```
kubectl -n rbac create role view --verb=list,watch,get --resource=role,rolebinding,pod
kubectl -n rbac create rolebinding view --role=view --serviceaccount=rbac:privesc
kubectl -n rbac create role edit --verb=update,patch,create --resource=rolebinding,pod
kubectl -n rbac create rolebinding edit --role=edit --serviceaccount=rbac:privesc
```

Отдельно создадим роли для работы с подами, но пока не будем их биндить:
```
kubectl -n rbac create role pod-view-edit --verb=get,list,watch,update,patch --resource=pod
kubectl -n rbac create role delete-pod --verb=delete --resource=pod
```

Переключимся на конфиг сервисаккаунта privesc и проверим, что мы можем редактировать rolebinding:
```
export KUBECONFIG=~/.kube/rbac.conf
kubectl -n rbac create rolebinding pod-view-edit --role=pod-view-edit --serviceaccount=rbac:privesc
rolebinding.rbac.authorization.k8s.io/pod-view-edit created
```
Новая роль успешно забиндилась к существующему сервисаккаунту. Обратите внимание, что роль `pod-view-edit` содержит `verbs` и `resources` уже подключенные сервисаккаунту в rolebinding `view` и `edit`.
Теперь попробуем забиндить роль с новым verb `delete`, которого ещё нет в уже подключенных к сервисаккаунту ролях:

```
kubectl -n rbac create rolebinding delete-pod --role=delete-pod --serviceaccount=rbac:privesc
error: failed to create rolebinding: rolebindings.rbac.authorization.k8s.io "delete-pod" is forbidden: user "system:serviceaccount:rbac:privesc" (groups=["system:serviceaccounts" "system:serviceaccounts:rbac" "system:authenticated"]) is attempting to grant RBAC permissions not currently held:
{APIGroups:[""], Resources:["pods"], Verbs:["delete"]}
```

Kubernetes не позволяет нам это сделать, несмотря на то, что у нас есть права на редактирование и создание RoleBindings. Исправить это нам поможет право `bind`. Используя админский конфиг выдадим его сервисаккаунту:
```
KUBECONFIG=~/.kube/config kubectl -n rbac create role bind --verb=bind --resource=role       
role.rbac.authorization.k8s.io/bind created

KUBECONFIG=~/.kube/config kubectl -n rbac create rolebinding bind --role=bind --serviceaccount=rbac:privesc
rolebinding.rbac.authorization.k8s.io/bind created
``` 

И повторим попытку создать RoleBinding с новым verb `delete`:
```
kubectl -n rbac create rolebinding delete-pod --role=delete-pod --serviceaccount=rbac:privesc
rolebinding.rbac.authorization.k8s.io/delete-pod created
```

Теперь RoleBinding создается.


### Impersonate
[DOC](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#user-impersonation)
Impersonation requests first authenticate as the requesting user, then switch to the impersonated user info.
- A user makes an API call with their credentials and impersonation headers.
- API server authenticates the user.
- API server ensures the authenticated users have impersonation privileges.
- Request user info is replaced with impersonation values.
- Request is evaluated, authorization acts on impersonated user info.

Impersonate это такой sudo для k8s. С правами impersonate пользователь может представиться другим пользователем и выполнять команды от его имени. kubectl имеет опции `--as`, `--as-group`, `--as-uid`, позволяющие выполнить команду от имени юзера, группы или uid соответственно. То есть, если в одной из ролей пользователь получил impersonate, то его можно считать админом неймспейса. Или кластера в худшем варианте - когда в неймспейсе есть ServiceAccount с правами cluster-admin.

Имперсонэйт удобно использовать для проверки корректности RBAC - запускать `kubectl auth can-i --as=$USERNAME -n $MANESPACE $VERB $RESOURCE`
```
kubectl auth can-i get pod -n rbac --as=system:serviceaccount:rbac:privesc
yes
```
Или вывести все права пользователя:
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

Хорошие виндовые админы должны помнить, что даже главные админы должны использовать аккаунты с минимальными правами для ежедневных дел, а важные консоли запускать от имени другого юзера с бОльшим количеством прав. В линуксе тот же принцип обеспечивается с помощью `sudo`. Так и в кубернетес - для защиты от случайного удаления важных данных, лучше не разрешать это действие обычному юзеру, а разрешить только админу, а юзеру дать права на impersonate, чтобы он мог использовать `--as=` когда это действительно нужно.

![impers](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*8XFhTnrLK8xLRr-eNl1PZw.png) НКАРИСОВАТЬ!

Создадим новый сервисаккаунт в неймспейсе rbac с именем impersonator, роль с правами impersonate и забиндим её на новый сервисаккаунт. Обратите внимание, что в роли мы разрешаем представляться только сервисаккаунтом `privesc` и никаким другим:
```
KUBECONFIG=~/.kube/config kubectl -n rbac create sa impersonator
serviceaccount/impersonator created

KUBECONFIG=~/.kube/config kubectl -n rbac create role impersonate --resource=serviceaccounts --verb=impersonate --resource-name=privesc
role.rbac.authorization.k8s.io/impersonate created

KUBECONFIG=~/.kube/config kubectl -n rbac create rolebinding impersonator --role=impersonate --serviceaccount=rbac:impersonator
rolebinding.rbac.authorization.k8s.io/impersonator created
```

Создадим новый контекст и проверим права нового сервисаккаунта:
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

https://github.com/postfinance/kubectl-sudo
проверить 
```yaml
# Can impersonate the user "jane.doe@example.com"
- apiGroups: [""]
  resources: ["users"]
  verbs: ["impersonate"]
  resourceNames: ["jane.doe@example.com"]
```


## Mitigation
Система авторизации k8s очень гибкая и позволяет гранулярно настраивать параметры доступа. Даже в тех случаях, когда пользователю необходимо управлять такими чувствительными для безопасности примитивами как Role/ClusterRole и RoleBinding/ClusterRoleBinding. При этом пользователь не сможет повысить свои привилегии и получить доступ к закрытым данным, если администратор явно не позволит ему повышать привилегии.

Повысить привилегии позволяют verbs `escalate`, `bind` и `impersonate`. Первый разрешает добавлять новые записи в Role/ClusterRole, второй - в RoleBinding/ClusterRoleBinding, а третий - представляться другим пользователем. Это очень мощные инструменты, неправильное использование которых может нанести значительный урон работе кластера. Следует очень внимательно проверять любое использование этих verbs и всегда убеждаться в том, что соблюдается правило Least Privilegies - пользователь должен иметь минимальный набор привилегий, необходимый для работы.

Для ограничения использования этих или любых других правил в манифестах Role/ClusterRole есть поле `resourceNames`, куда можно (и нужно!) вписать имена ресурсов, которые можно использовать.
В этом примере пользователю разрешено создавать ClusterRoleBinding с `roleRef` с именами "admin","edit","view":
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

То же самое можно сделать с escalate или impersonate. С `bind` права в роли задает админ, а пользователь только может забиндить эту роль на себя, если это разрешено в `resourceNames`, то с `escalate` пользователь может внутри роли прописать любые параметры и стать админом неймспейса или кластера. То есть `escalate` дает больше возможностей, в то время как `bind` ограничивает пользователя. Имейте это в виду, когда приедтся давать эти права пользователям.

Для контроля за этими опасными нюансами следует периодически проверять их наличие в кластере:
```
kubectl get clusterrole -A -oyaml | yq '.items[] | select (.rules[].verbs[] | contains("esalate" | "bind" | "impersonate"))  | .metadata.name'

kubectl get role -A -oyaml | yq '.items[] | select (.rules[].verbs[] | contains("esalate" | "bind" | "impersonate"))  | .metadata.name'
```
Или настроить автоматизированные системы, следящие за созданием или редактированием ролей с подозрительным содержанием, например Falco СЮДА МОЖНО ПРИЛЕПИТЬ РЕКЛАМУ НАШЕГО Logs as a Service


Kubernetes предоставляет очень гибкие возможности для управления правами доступа. Вместе с гибкостью приходят и сложность поддержки наряду с угрозами безопасности. Для обеспечения надежной и безопасной работы нужно понимать принципы функционирования системы авторизации и типы вспомогательного инструментария, снижающего риски. И помните, что безопасность всей системы равна безопасности её самого слабого звена.



### Tools
НАПИСАТЬ КУДА_ТО о том, что если есть серт в кофниге, то токен не используется!! https://stackoverflow.com/questions/60083889/kubectl-token-token-doesnt-run-with-the-permissions-of-the-token

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




k get clusterrole -A -oyaml | yq '.items[] | select (.rules[].verbs[] | contains("esalate" | "bind" | "impersonate"))  | .metadata.name'