Azure AD, PAM, OVPN plugins
https://medium.com/@jkroepke/openvpn-sso-via-oauth2-ab2583ee8477
https://blog.please-open.it/openvpn-keycloak/
https://medium.com/@hiranadikari993/openvpn-active-directory-authentication-726f3bac3546
https://github.com/threerings/openvpn-auth-ldap
https://community.openvpn.net/openvpn/wiki/PluginOverview
https://github.com/OpenVPN/openvpn/blob/master/src/plugins/auth-pam/README.auth-pam
https://github.com/ubuntu/aad-auth
https://github.com/aad-for-linux/openvpn-auth-aad
И PDF PAM Guide ...


keycloak - мониторинг, kuberos, сборрка kuberos без CVE
"serviceaccounts" - запросить id_token
TOKEN=$(curl \
  -d "client_id=CLIENT_ID" -d "client_secret=CLIENT_SECRET" \
  -d "username=bserviceaccount" -d "password=1234567890" \
  -d "grant_type=password" \
  -d "scope=openid" \
  "https://KEYCLOAK/realms/master/protocol/openid-connect/token" | jq -r '.id_token')

Потом пойти с этим токеном в куб KUBECONFIG=~/Downloads/dev1.conf k --token $TOKEN get pod 
в выводе токена должен быть поле name - то поле, которое указали аписерверу oidc-username-claim: name

в клок добавить что если уже авторизовалмя в одном клиенте и пошел в другой, то клок может отдавать 502. Это не клок виноват. Логи ингреса " upstream sent too big header while reading response header from upstream". Надо увеличить proxy_buffers and proxy_buffers_size через аннтиоции    annotations:
    nginx.ingress.kubernetes.io/proxy-buffer-size: "16k"

проверка авторизации клок 
kubectl config set-credentials oidc \
  --exec-api-version=client.authentication.k8s.io/v1beta1 \
  --exec-command=kubectl \
  --exec-arg=oidc-login \
  --exec-arg=get-token \
  --exec-arg=--oidc-issuer-url=https://keycloak.k-stg-1.luxembourg-2.cloud.gc.onl/realms/realmWithLdap \
  --exec-arg=--oidc-client-id=ed-16-k-stg-1 \
  --exec-arg=--oidc-client-secret=нделепалклаплка




kubectl oidc-login setup --oidc-issuer-url https://keycloak.k-stg-1.luxembourg-2.cloud.gc.onl/realms/realmWithLdap --oidc-client-id ed-16-k-stg-1 --oidc-client-secret ун4цугн3ун5у4г



https://kubernetes.io/docs/reference/access-authn-authz/authentication/#user-impersonation

https://rad.security/blog/what-is-kubernetes-rbac#RBACpolice
# Topic
- What is the general topic you want to cover?
impersonate is urgent not so popular feature 
- Do you already have a potential title in mind? (If not, come up with 3 viable options together.)
Some non-obvious features  Kubernetes RBAC, Impersonate it
- Are there any particular subtopics you would like to focus on? (Outline specific headings together.)
bellow
- Are there any relevant topics that should be avoided?
no
- Are there any existing pieces of content that you would like us to reference or align with?
k8s documentation
# Audience
- Who is the primary target audience for this content? What job titles do they have?
devops jun+/middle
- Are there any secondary audiences we should keep in mind?
developers, devops junior
- Is the reader a beginner, intermediate, or expert?
intermediate
- What value do you hope the audience will gain from this content? What is the key takeaway?
know about nonobvious features of k8s rbac
- Are there any common questions or concerns of the audience that we should address?
# Purpose
- What is the primary purpose or goal of this content? (E.g., new product awareness, existing product purchase, increase traffic to product page)
threat awareness 
- Is there a secondary purpose or goal we should consider?
better understanding of k8s rbac
- Are you hoping to promote a specific product or service through this content?
managed kubernetes
- How will you measure the success of this content?
likes and reposts
# Length and Genre
- Do you have a preferred length for this content?
7-10 minutes read
- Is there a specific genre or style you're aiming for (blog post, white paper, case study, etc.)?

- Are there any visual elements (graphs, charts, images, etc.) that need to be included?
images, code


- inro
- rbac
- impersonate
- угрозы
- как защититься
- обзор инструментов

Ещё одна угроза - Фggregated ClusterRole 
[Хорошая презентация про RBAC](https://www.cncf.io/wp-content/uploads/2020/08/2020_04_Introduction-to-Kubernetes-RBAC.pdf), [видео](https://www.youtube.com/watch?v=B6Ylwugs3t0)

На первый взляд система RBAC в k8s достаточно простая и безопасная, но многие забывают или не знают некоторых осоюенностей
В kubernetes есть аналог sudo, благодаря которому можно обойти ограничения в назначенных ролях и получить доступ туда, куда не предполагалось и прочитать скеретные данные или даже получить полный контроль над кластером. И эта возможность есть у любого юзера с дефолтной кластерролью edit. В этой статье попробуем разобраться что это такое и как защититься от такого легального повышения привилегий.

## Ролевая модель доступа в Kubernetes

Cуществует несколько подходов к разграничению доступа к ресурсам:
- ABAC - Attribute Based Access Control
- RBAC - Role Based Access Control
- PBAC - Policy Based Access Control
- DAC - Discretionary Access Control
- MAC - Mandatory Access Control
- MLS/MCS - Multi Level Security /  Multi Category Security

Kubernetes может использовать модели доступа: [Node, ABAC, RBAC, WebHook](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#authorization-modules).
Чаще всего приходится работать с ролевой моделью доступа - RBAC, т.к. она используется в k8s по умлочанию. В ней есть несколько взаимосвязанных сущностей:
- роль, описывающая доступ к ресурсам
- пользователь (это может быть User, Group или ServiceAccount)
- Rolebinding - то, что объединяет роли и пользователей 

На самом деле этих сущностей немного больше. Рассмотрим их детальней.
Про RBAC хорошо написано в [документации k8s](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) и я не буду пытаться написать ещё раз. Просто рассмотрим то, без чего остальная статья не имеет смысла. Если читатель знает концепции RBAC k8s - эту часть можно пропусать и сразу переходить к следующей НАЗВАНИЕ ЧАСТИ ТУТ

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

## Неочевидные нюансы k8s RBAC
Наконец-то добрались до того, ради чего писался этот пост.
Не самые популярные, но опасные нюансы в RBAC.

### Escalate
[DOC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#restrictions-on-role-creation-or-update): You can only create/update a _role_ if at least one of the following things is true:
- You already have all the permissions contained in the role, at the same scope as the object being modified (cluster-wide for a ClusterRole, within the same namespace or cluster-wide for a Role).
- You are granted explicit permission to perform the `escalate` verb on the roles or clusterroles resource in the rbac.authorization.k8s.io API group.

[Kubernetes RBAC API не позволяет повысить привелегии доступа путем редактирования Role или RoleBinding. Это происходит на уровне API и будет работать даже если выключен RBAC authorizer](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#privilege-escalation-prevention-and-bootstrapping). Исключение из этого правила - наличие права `escalate` у роли.
![escalate](/img/k8s-rbac-nuances/escalate-purple-white.png) 

Роль с правами на просмотр подов и ролей в неймспейсе rbac:
```
kubectl -n rbac create role view --verb=list,watch,get --resource=role,pod
role.rbac.authorization.k8s.io/view created 
```

Биндим созданный ранее сервисаккаунт к нашей роли:
```
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

Сделаем роль с правами на редактирование ролей в неймспейсе rbac:
```
kubectl -n rbac create role edit --verb=update,patch --resource=role                              
role.rbac.authorization.k8s.io/edit created
```

Биндим к нашему сервисаккаунту:
```
kubectl -n rbac create rolebinding edit --role=edit --serviceaccount=rbac:privesc
rolebinding.rbac.authorization.k8s.io/edit created
```

Проверим:
```
kubectl auth can-i update role -n rbac --as=system:serviceaccount:rbac:privesc   
yes

kubectl auth can-i delete role -n rbac --as=system:serviceaccount:rbac:privesc
no
```

Теперь для чистоты эксперимента проверим возможности нашего сервисаккаунта. Для этого используем его токен в конфиге. 
`TOKEN=$(kubectl -n rbac create token privesc --duration=8h)`

Придется удалить из конфига старые параметры аутентификации, т.к. [k8s сначала проверяет сертификат пользователя и не будет проверять токен, если данные о сертификате переданы](https://stackoverflow.com/questions/60083889/kubectl-token-token-doesnt-run-with-the-permissions-of-the-token).
```
cp ~/.kube/config ~/.kube/rbac.conf
export KUBECONFIG=~/.kube/rbac.conf
kubectl config delete-user kubernetes-admin
kubectl config set-credentials privesc --token=$TOKEN
kubectl config set-context --current --user=privesc
```

В роли edit написано, что мы имеем права не редактирование ролей:
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

Попробуем добавить в роль новый verb (list), который уже имеется в другой роли (view)
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
  - list   # <-- добавлена эта строка
```

Попробуем добавить в роль новый verb (delete), который не описан в других ролях:
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
  - delete   # <-- добавлена эта строка

error: roles.rbac.authorization.k8s.io "edit" could not be patched: roles.rbac.authorization.k8s.io "edit" is forbidden: user "system:serviceaccount:rbac:privesc" (groups=["system:serviceaccounts" "system:serviceaccounts:rbac" "system:authenticated"]) is attempting to grant RBAC permissions not currently held:
{APIGroups:["rbac.authorization.k8s.io"], Resources:["roles"], Verbs:["delete"]}
```
Kubernetes не позволяет добавлять себе новых прав, которых ещё нет у пользователя - прав, которые не описаны в других ролях, забинденых к этому пользователю

Используя админские права расширим права сервисаккаунта privesc - добавим новую роль с verb escalate и забиндим её нашему сервисаккаунту:
```
KUBECONFIG=~/.kube/config kubectl -n rbac create role escalate --verb=escalate --resource=role   
role.rbac.authorization.k8s.io/escalate created

KUBECONFIG=~/.kube/config kubectl -n rbac create rolebinding escalate --role=escalate --serviceaccount=rbac:privesc
rolebinding.rbac.authorization.k8s.io/escalate created
```

Повторяем то, что не удалось сделать без escalate - добавить delete в существующую роль:
```
kubectl -n rbac edit  role edit 
role.rbac.authorization.k8s.io/edit edited
```

Теперь это работает. Пользователь может повышать свои привилегии редактируя существующие роли. То есть verb escalate фактически дает права администратора, т.к. пользователь, обладающий привилегиями escalate может выписать себе любые права на неймспейс или кластер, если это указано в resources.

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