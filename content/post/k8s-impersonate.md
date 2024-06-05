






- inro
- rbac
- impersonate
- угрозы
- как защититься
- обзор инструментов

Ещё одна угроза - Фggregated ClusterRole 

На первый взляд система RBAC в k8s достаточно простая и безопасная, но многие забывают или не знают некоторых особенностей.

В kubernetes есть аналог sudo, благодаря которому можно обойти ограничения в назначенных ролях и получить доступ туда, куда не предполагалось и прочитать секретные данные или даже получить полный контроль над кластером. И эта возможность есть у любого юзера с дефолтной кластерролью edit. В этой статье попробуем разобраться что это такое и как защититься от такого легального повышения привилегий.


## Ресурсы kubernetes RBAC
Работа системы аутентификации была подробно описана в [первой части](https://bubnovd.net/post/k8s-auth/). Рекомендую прочитать её сначала. Тут очень кратко о том, что пригодится дальше.
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



### Escalate
[DOC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#restrictions-on-role-creation-or-update): You can only create/update a _role_ if at least one of the following things is true:
- You already have all the permissions contained in the role, at the same scope as the object being modified (cluster-wide for a ClusterRole, within the same namespace or cluster-wide for a Role).
- You are granted explicit permission to perform the `escalate` verb on the roles or clusterroles resource in the rbac.authorization.k8s.io API group.

[Kubernetes RBAC API не позволяет повысить привелегии путем редактирования Role или RoleBinding. Это происходит на уровне API и будет работать даже если выключен RBAC authorizer](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#privilege-escalation-prevention-and-bootstrapping). Исключение из этого правила - наличие права `escalate` у роли. То есть МОЖНО создавать и редактировать роли, но НЕЛЬЗЯ добавлять юзерам новые права - можно оперировать лишь теми правами, которые уже есть у юзера в других ролях
![escalate](/img/k8s-rbac-nuances/escalate-purple-white.png)

Проверим как это работает:

Для начала создадим неймспейс и сервисаккаунт, с которыми будем работать:
```bash
kubectl create ns rbac
kubectl create sa -n rbac privesc
```

Создадим роль `view` с правами только на чтение и роль `edit` с правами на редактирование и проверим может ли пользователь повысить себе привилегии, добавляя новые verbs в роли.

Создаём роль с правами на просмотр ролей в неймспейсе `rbac` и биндим её к SA:
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

Дадим права на редактирование ролей в неймспейсе `rbac`:
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

Теперь для чистоты эксперимента проверим возможности нашего сервисаккаунта, залогинившись в k8s от его имени. Для этого используем его токен в конфиге. Как это работает - смотрите в [первой части](https://bubnovd.net/post/k8s-auth/). 
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

Работает!

Теперь попробуем добавить в роль новый verb `delete`, который не описан в других ролях:
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
Kubernetes не позволяет добавлять новых прав, которых ещё нет у пользователя - прав, которые не описаны в других ролях, забинденых к этому пользователю. Попробуем обойти это.

Как мы убедились в последнем примере - повысить привилегии самому себе нельзя, поэтому дальнейшие действия будем делать от админа - об этом говорит задание переменной `KUBECONFIG=~/.kube/config` перед командой. Расширим права сервисаккаунта privesc - добавим новую роль с verb `escalate` и забиндим её нашему сервисаккаунту:
```bash
KUBECONFIG=~/.kube/config kubectl -n rbac create role escalate --verb=escalate --resource=role   
role.rbac.authorization.k8s.io/escalate created

KUBECONFIG=~/.kube/config kubectl -n rbac create rolebinding escalate --role=escalate --serviceaccount=rbac:privesc
rolebinding.rbac.authorization.k8s.io/escalate created
```

Теперь с уже расширенными правами повторяем то, что не удалось сделать без `escalate` - добавить `delete` в существующую роль:
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
Теперь это работает. Пользователь может повышать свои привилегии редактируя существующие роли. То есть verb `escalate` фактически дает права администратора, т.к. пользователь, обладающий привилегиями `escalate` может выписать себе любые права на неймспейс или даже кластер.

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

`Bind` работает аналогично `escalate`. Если `escalate` разрешает редактировать `Role` или `ClusterRole` для повышения привилегий, то `bind` разрешает редактировать `RoleBinding` или `ClusterRoleBinding`. 
Если прав в роли недостаточно, то пользователь может забиндить себя на другую роль.
![pic](/img/k8s-rbac-nuances/bind-purple-white.png)

Переключимся на админский конфиг: `export KUBECONFIG=~/.kube/config`

Удалим созданные ранее роли и биндинги
```bash
kubectl -n rbac delete rolebinding view edit escalate
kubectl -n rbac delete role view edit escalate
```

Дадим сервисаккаунту права на просмотр и редактирование rolebinding и pod в своём неймспейсе:
```bash
kubectl -n rbac create role view --verb=list,watch,get --resource=role,rolebinding,pod
kubectl -n rbac create rolebinding view --role=view --serviceaccount=rbac:privesc
kubectl -n rbac create role edit --verb=update,patch,create --resource=rolebinding,pod
kubectl -n rbac create rolebinding edit --role=edit --serviceaccount=rbac:privesc
```

Отдельно создадим роли для работы с подами, но пока не будем их биндить. Нам нужны эти роли, чтобы проверить способность сервисакаунта биндить новые роли:
```bash
kubectl -n rbac create role pod-view-edit --verb=get,list,watch,update,patch --resource=pod
kubectl -n rbac create role delete-pod --verb=delete --resource=pod
```

Переключимся на конфиг сервисаккаунта privesc и проверим, что мы можем редактировать rolebinding:
```bash
export KUBECONFIG=~/.kube/rbac.conf
kubectl -n rbac create rolebinding pod-view-edit --role=pod-view-edit --serviceaccount=rbac:privesc
rolebinding.rbac.authorization.k8s.io/pod-view-edit created
```
Новая роль успешно забиндилась к существующему сервисаккаунту. Обратите внимание, что роль `pod-view-edit` содержит `verbs` и `resources` уже подключенные сервисаккаунту в rolebinding `view` и `edit`.
Теперь попробуем забиндить роль с новым verb `delete`, которого ещё нет в уже подключенных к сервисаккаунту ролях:

```bash
kubectl -n rbac create rolebinding delete-pod --role=delete-pod --serviceaccount=rbac:privesc
error: failed to create rolebinding: rolebindings.rbac.authorization.k8s.io "delete-pod" is forbidden: user "system:serviceaccount:rbac:privesc" (groups=["system:serviceaccounts" "system:serviceaccounts:rbac" "system:authenticated"]) is attempting to grant RBAC permissions not currently held:
{APIGroups:[""], Resources:["pods"], Verbs:["delete"]}
```

Kubernetes не позволяет нам это сделать, несмотря на то, что у нас есть права на редактирование и создание RoleBindings. Исправить это нам поможет право `bind`. Используя админский конфиг выдадим его сервисаккаунту:
```bash
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

Теперь RoleBinding создается. Verb `bind` позволяет биндить любые роли и повышать привилегии.


### Impersonate
[DOC](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#user-impersonation)
Impersonation requests first authenticate as the requesting user, then switch to the impersonated user info.
- A user makes an API call with their credentials and impersonation headers.
- API server authenticates the user.
- API server ensures the authenticated users have impersonation privileges.
- Request user info is replaced with impersonation values.
- Request is evaluated, authorization acts on impersonated user info.

`Impersonate` это такой `sudo` для k8s. С правами `impersonate` пользователь может представиться другим пользователем и выполнять команды от его имени. kubectl имеет опции `--as`, `--as-group`, `--as-uid`, позволяющие выполнить команду от имени юзера, группы или uid соответственно. То есть, если в одной из ролей пользователь получил `impersonate`, то его можно считать админом неймспейса. Или кластера в худшем варианте - когда в неймспейсе есть ServiceAccount с правами cluster-admin. ТОЧНО ТАК??

Имперсонэйт удобно использовать для проверки корректности RBAC - запускать `kubectl auth can-i --as=$USERNAME -n $NAMESPACE $VERB $RESOURCE`
```bash
kubectl auth can-i get pod -n rbac --as=system:serviceaccount:rbac:privesc
yes
```
Или вывести все права пользователя:
```bash
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

Хорошие виндовые админы должны помнить, что даже главные админы должны использовать аккаунты с минимальными правами для ежедневных дел, а важные консоли запускать от имени другого юзера с бОльшим количеством прав. В линуксе тот же принцип обеспечивается с помощью `sudo`. Так и в кубернетес - для защиты от случайного удаления важных данных, лучше не разрешать это действие обычному юзеру - разрешить только админу, а юзеру дать права на `impersonate`, чтобы он мог использовать `--as=` когда это действительно нужно.

![impersonate](/img/k8s-rbac-nuances/impersonate-white.png)

Создадим новый сервисаккаунт в неймспейсе `rbac` с именем `impersonator`, роль с правами `impersonate` и забиндим её на новый сервисаккаунт. Обратите внимание, что в роли мы разрешаем представляться только сервисаккаунтом `privesc` и никаким другим:
```bash
KUBECONFIG=~/.kube/config kubectl -n rbac create sa impersonator
serviceaccount/impersonator created

KUBECONFIG=~/.kube/config kubectl -n rbac create role impersonate --resource=serviceaccounts --verb=impersonate --resource-name=privesc
role.rbac.authorization.k8s.io/impersonate created

KUBECONFIG=~/.kube/config kubectl -n rbac create rolebinding impersonator --role=impersonate --serviceaccount=rbac:impersonator
rolebinding.rbac.authorization.k8s.io/impersonator created
```

Создадим новый контекст и проверим права нового сервисаккаунта:
```bash
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

Ничего лишнего нет - только `impersonate`, как и указано в роли. Но если воспользоваться правом `impersonate` и представиться как сервисаккаунт `privesc`, то видно, что права совпадают с сервисаккаунтом `privesc`:
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
То есть с `impersonate` сервисаккаунт получил все свои права + все права сервисаккаунта, которым он может представляться, фактически получив неограниченный доступ к неймспейсу.

https://github.com/postfinance/kubectl-sudo
проверить 
```yaml
# Can impersonate the user "jane.doe@example.com"
- apiGroups: [""]
  resources: ["users"]
  verbs: ["impersonate"]
  resourceNames: ["jane.doe@example.com"]
```

### Aggregated ClusterRoles
Помимо "стандартных" ресурсов (`pod`, `secret`, `service`, ...) в кластере могут появляться дополнительные ресурсы, описываемые CRD (`certificates`, `kafkas`, `prometheuses`, ...). Очевидно, что в дефолтных ролях из коробки невозможно предусмотреть ресурсы, которые будут установлены в будущем через CRD и дать необходимые привилегии для управления ими. Для работы с этим ограничением существует Aggregated ClusterRole - она аггрегирует в себе правила из других ролей, при этом сама может ни одного правила не содержать. Примеры таких ролей - дефолтные admin, view, edit. Все они содержат параметр `aggregationRule` c лейблами, по которым идентифируются кластерроли, правила из которых будут агрегированы в целевую кластерроль.

`k get clusterrole admin -oyaml`
```yaml
aggregationRule:
  clusterRoleSelectors:
    - matchLabels:
        rbac.authorization.k8s.io/aggregate-to-admin: "true"
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  name: admin
...
```

То есть в роли `admin` будут все правила, явно описанные в `admin`, а так же все правила всех ролей с лэйблом `rbac.authorization.k8s.io/aggregate-to-admin: "true"`. Например, `cert-manager-view`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-view: "true"
  name: cert-manager-view
rules:
  - apiGroups:
      - cert-manager.io
    resources:
      - certificates
      - certificaterequests
      - issuers
    verbs:
      - get
      - list
      - watch
...
```

Хоть дефолтные роли `admin`, `view` и `edit` ничего не знают о сущности `certificates`, после создания кластерроли `cert-manager-view` контроллер `clusterrole-aggregation-controller` увидит, что роль содержит необходимые лейблы и роли с соттветствующим `aggregationRule` смогут манипулировать `certificates` и другими сущностями.
Важно тут то, что сама роль с `aggregationRule` - `admin` в нашем случае - никак не изменится. Дополнительные правила в неё не добавятся и обычным `kubectl get clusterrole admin -o yaml` эти привелегии не увидеть. Что создает возможности для злоумышленника или проблемы для админа. Некоторые Aggregated ClusterRole могут совсем не содержать никаких `rules`, а только `aggregationRule`. Хороший пример есть в [документации](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#aggregated-clusterroles).

## Mitigation
Система авторизации k8s очень гибкая и позволяет гранулярно настраивать параметры доступа. Даже в тех случаях, когда пользователю необходимо управлять такими чувствительными для безопасности примитивами как `Role`/`ClusterRole` и `RoleBinding`/`ClusterRoleBinding`. При этом пользователь не сможет повысить свои привилегии и получить доступ к закрытым данным, если администратор явно не позволит ему повышать привилегии.

Повысить привилегии позволяют verbs `escalate`, `bind` и `impersonate`. Первый разрешает добавлять новые записи в Role/ClusterRole, второй - в RoleBinding/ClusterRoleBinding, а третий - представляться другим пользователем. Это очень мощные инструменты, неправильное использование которых может нанести значительный урон работе кластера. Следует очень внимательно проверять любое использование этих verbs и всегда убеждаться в том, что соблюдается правило Least Privilegies - пользователь должен иметь минимальный набор привилегий, необходимый для работы.

Для ограничения использования этих или любых других правил в манифестах `Role`/`ClusterRole` есть поле `resourceNames`, куда можно (и нужно!) вписать имена ресурсов, которые можно использовать.
В этом примере пользователю разрешено создавать ClusterRoleBinding с `roleRef` с именами `admin`,`edit`,`view`:
```yaml
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

То же самое можно сделать с `escalate` или `impersonate`. Если с `bind` права в роли задает админ, а пользователь только может забиндить эту роль на себя, если это разрешено в `resourceNames`, то с `escalate` пользователь может внутри роли прописать любые параметры и стать админом неймспейса или кластера. То есть `escalate` дает больше возможностей, в то время как `bind` ограничивает пользователя. Имейте это в виду, когда приедтся давать эти права пользователям.

Для контроля за этими опасными нюансами следует периодически проверять их наличие в кластере:
```bash
kubectl get clusterrole -A -oyaml | yq '.items[] | select (.rules[].verbs[] | contains("esalate" | "bind" | "impersonate"))  | .metadata.name'

kubectl get role -A -oyaml | yq '.items[] | select (.rules[].verbs[] | contains("esalate" | "bind" | "impersonate"))  | .metadata.name'
```

Так же следует следить за появлением ролей с `aggregationRule` - такие роли, кажущиеся абсолютно невинными, могут иметь полные права на неймспейс:
`kubectl get clusterrole -A -oyaml | yq '.items[] | select (.aggregationRule) | .metadata.name'`

Или настроить автоматизированные системы, следящие за созданием или редактированием ролей с подозрительным содержанием, например [Falco](https://falco.org/) или [Tetragon](https://tetragon.io/).


[kubectl-who-can](https://github.com/aquasecurity/kubectl-who-can) позволяет увидеть все субъекты, способные выполнять определенные действия

---
Kubernetes предоставляет очень гибкие возможности для управления правами доступа. Несмотря на это, до сих пор не существует механизма отзыва сертификатов. То есть kubeconfig будет действовать ровно до того момента, пока не истек срок действия его сертификата, что заставляет беречь сертификаты с молоду, а не "потом разберемся". Об этом поговорим в следующей части.
