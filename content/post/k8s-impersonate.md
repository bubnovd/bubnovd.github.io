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


https://kubernetes.io/docs/reference/access-authn-authz/authentication/#user-impersonation


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

В kubernetes есть аналог sudo, благодаря которому можно обойти ограничения в назначенных ролях и получить доступ туда, куда не предполагалось. И эта возможность есть у любого юзера с дефолтной кластерролью edit. В этой статье попробуем разобраться что это такое и как защититься от такого легального повышения привилегий.

Зачем используетя? 
- для дебага рбака обычно


## Ролевая модель доступа в Kubernetes

Cуществует несколько подходов к разграничению доступа к ресурсам:
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

## Ресурсы kubernetes RBAC
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
- [escalate](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#escalate-verb) - создавать и редактировать роли ЧУЖИЕ?? 
- [bind](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#bind-verb) - создавать и редактировать биндинги ЧУЖИЕ?? 
- [impersonate](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#impersonate-verb) - представиться k8s API другим пользователем, группой или предоставить другие данные (extra)

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

#### JWT
Сгенерируем токен для аутентификации от имени SA:
`TOKEN=$(k create token mysa)`
СГЕНЕРИТЬ И ПОКАЗАТЬ
Получили JWT. 
> Обычно по-русски пишут JWT токен, что является [плеоназмом](https://ru.wikipedia.org/wiki/%D0%90%D0%B1%D0%B1%D1%80%D0%B5%D0%B2%D0%B8%D0%B0%D1%82%D1%83%D1%80%D0%B0#%D0%A2%D0%B0%D0%B2%D1%82%D0%BE%D0%BB%D0%BE%D0%B3%D0%B8%D1%87%D0%B5%D1%81%D0%BA%D0%BE%D0%B5_%D1%81%D0%BE%D0%BA%D1%80%D0%B0%D1%89%D0%B5%D0%BD%D0%B8%D0%B5): JSON Web Token токен. Поэтому я буду писать просто JWT

JWT состоит из трех частей, разделенных точками. Первые две части - закодированный в base64 текст, последняя - подпись.
`echo $TOKEN | jq -R ' split(".") | select(length > 0) | .[0],.[1] | @base64d | fromjson'`
ПОКАЗАТЬ ЧТО ПОЛУЧИЛОСЬ

#### Users, Groups

Это может показаться странным, но API k8s не имеет понятия юзера или группы. Невозможно создать пользователя или группу. Но эти данные необходимы для авторизации. API Server распознает пользователя по полю CN в Subject сертификата.
ПОКАЗАТЬТ КАК openssl x509 -noout -text -in ~/gcore/tmp/c.crt  

В качестве системы аутентификации k8s может использовать сторонние Identity Providers и интегрироваться с ними по OpenID Connect. Например, keycloak (ССЫЛКА ТУТ)

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