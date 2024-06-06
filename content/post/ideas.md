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

пост про bastion/bastillion

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


Вместе с гибкостью приходят и сложность поддержки наряду с угрозами безопасности. Для обеспечения надежной и безопасной работы нужно понимать принципы функционирования системы авторизации и типы вспомогательного инструментария, снижающего риски. И помните, что безопасность всей системы равна безопасности её самого слабого звена.


В следующей части 
НАПИСАТЬ ТУТ ЧТО БУДЕТ В СЛЕДУЩЕЙ АСТИ:
- CA
- keycloak OIDC
- piopeline OIDC & rbac-definition
- bastion/bastillion