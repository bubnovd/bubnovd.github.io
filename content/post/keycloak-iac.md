+++
title = "Keycloak. Configuration as Code"
date = "2023-11-22T04:02:52Z"
author = "bubnovd"
authorTwitter = "bubnovdnet"
cover = ""
tags = ["keycloak", "iac"]
keywords = ["keycloak", "iac", "bitnami", "chart"]
description = "Configuration as Code for Keycloak chart"
showFullContent = false
readingTime = true
hideComments = false
+++

Представленная конфигурация keycloak создает федерацию LDAP для подключения к Active Directory и клиента k8s-client для взаимодействия с kubernetes по OIDC. В федерацию добавлен маппинг групп AD.

Конфигурацию производит job `keycloak-config-cli` после деплоя хелм чарта (post-install hook). [Детальное описание](https://github.com/adorsys/keycloak-config-cli), [примеры конфигурации](https://github.com/adorsys/keycloak-config-cli/tree/main/src/test/resources/import-files)

В kube-apiserver нужно добавить аргументы:

    --oidc-client-id=k8s-client
    --oidc-issuer-url=https://your.keycloak.url/realms/realmWithLdap
    --oidc-groups-claim=groups
    --oidc-username-claim=name

    следующие аргументы необязательны
    --oidc-username-prefix="keycloak:"
    --oidc-groups-prefix="keycloak:"

Конфигурация работает с чартом https://github.com/bitnami/charts/tree/main/bitnami/keycloak версии 17.1.0

- Secret `keycloak-config-cli` must contain CLIENT_SECRET and LDAP_BIND_CREDS
- `IMPORT_VARSUBSTITUTION_ENABLED` needs for substitute data from secret like `"$(env:LDAP_BIND_CREDS)"`

```
keycloakConfigCli:
    enabled: true
    image:
      registry: docker.io
      repository: bitnami/keycloak-config-cli
      tag: 5.9.0-debian-11-r0
  
    extraEnvVars:
     - name: IMPORT_VARSUBSTITUTION_ENABLED
       value: 'true'
  
    # secret must contain: CLIENT_SECRET and LDAP_BIND_CREDS
    extraEnvVarsSecret: "keycloak-config-cli"
    configuration:
      realm1.json: |
        {
          "enabled": true,
          "realm": "realmWithLdap",
          "components": {
            "org.keycloak.storage.UserStorageProvider": [
              {
                "id": "ldap",
                "name": "ldap",
                "providerId": "ldap",
                "subComponents": {
                  "org.keycloak.storage.ldap.mappers.LDAPStorageMapper": [
                    {
                      "name": "last name",
                      "providerId": "user-attribute-ldap-mapper",
                      "subComponents": {},
                      "config": {
                        "is.mandatory.in.ldap": [
                          "true"
                        ],
                        "read.only": [
                          "true"
                        ],
                        "always.read.value.from.ldap": [
                          "true"
                        ],
                        "user.model.attribute": [
                          "lastName"
                        ]
                      }
                    },
                    {
                      "name": "first name",
                      "providerId": "user-attribute-ldap-mapper",
                      "subComponents": {},
                      "config": {
                        "ldap.attribute": [
                          "cn"
                        ],
                        "is.mandatory.in.ldap": [
                          "true"
                        ],
                        "always.read.value.from.ldap": [
                          "true"
                        ],
                        "read.only": [
                          "true"
                        ],
                        "user.model.attribute": [
                          "firstName"
                        ]
                      }
                    },
                    {
                      "name": "creation date",
                      "providerId": "user-attribute-ldap-mapper",
                      "subComponents": {},
                      "config": {
                        "ldap.attribute": [
                          "createTimestamp"
                        ],
                        "is.mandatory.in.ldap": [
                          "false"
                        ],
                        "always.read.value.from.ldap": [
                          "true"
                        ],
                        "read.only": [
                          "true"
                        ],
                        "user.model.attribute": [
                          "createTimestamp"
                        ]
                      }
                    },
                    {
                      "name": "username",
                      "providerId": "user-attribute-ldap-mapper",
                      "subComponents": {},
                      "config": {
                        "ldap.attribute": [
                          "cn"
                        ],
                        "is.mandatory.in.ldap": [
                          "true"
                        ],
                        "always.read.value.from.ldap": [
                          "false"
                        ],
                        "read.only": [
                          "true"
                        ],
                        "user.model.attribute": [
                          "username"
                        ]
                      }
                    },
                    {
                      "name": "modify date",
                      "providerId": "user-attribute-ldap-mapper",
                      "subComponents": {},
                      "config": {
                        "ldap.attribute": [
                          "modifyTimestamp"
                        ],
                        "is.mandatory.in.ldap": [
                          "false"
                        ],
                        "read.only": [
                          "true"
                        ],
                        "always.read.value.from.ldap": [
                          "true"
                        ],
                        "user.model.attribute": [
                          "modifyTimestamp"
                        ]
                      }
                    },
                    {
                      "name": "email",
                      "providerId": "user-attribute-ldap-mapper",
                      "subComponents": {},
                      "config": {
                        "ldap.attribute": [
                          "mail"
                        ],
                        "is.mandatory.in.ldap": [
                          "false"
                        ],
                        "always.read.value.from.ldap": [
                          "false"
                        ],
                        "read.only": [
                          "true"
                        ],
                        "user.model.attribute": [
                          "email"
                        ]
                      }
                    },
                    {
                      "name": "ldap-groups",
                      "providerId": "group-ldap-mapper",
                      "subComponents": {},
                      "config": {
                        "membership.attribute.type": [
                          "DN"
                        ],
                        "group.name.ldap.attribute": [
                          "cn"
                        ],
                        "membership.user.ldap.attribute": [
                          "cn"
                        ],
                        "preserve.group.inheritance": [
                          "false"
                        ],
                        "groups.dn": [
                          "ou=YOUR_GROUPS_DN,dc=YOUR_DOMAIN,dc=YOUR_LOCAL"
                        ],
                        "mode": [
                          "READ_ONLY"
                        ],
                        "user.roles.retrieve.strategy": [
                          "GET_GROUPS_FROM_USER_MEMBEROF_ATTRIBUTE"
                        ],
                        "membership.ldap.attribute": [
                          "member"
                        ],
                        "ignore.missing.groups": [
                          "true"
                        ],
                        "group.object.classes": [
                          "group"
                        ],
                        "memberof.ldap.attribute": [
                          "memberOf"
                        ],
                        "groups.path": [
                          "/"
                        ],
                        "drop.non.existing.groups.during.sync": [
                          "true"
                        ]
                      }
                    }
                  ]
                },
                "config": {
                  "pagination": [
                    "false"
                  ],
                  "fullSyncPeriod": [
                    "30000"
                  ],
                  "connectionPooling": [
                    "true"
                  ],
                  "usersDn": [
                    "ou=YOUR_USERS_DN,dc=YOUR_DOMAIN,dc=YOUR_LOCAL"
                  ],
                  "cachePolicy": [
                    "DEFAULT"
                  ],
                  "useKerberosForPasswordAuthentication": [
                    "false"
                  ],
                  "importEnabled": [
                    "true"
                  ],
                  "enabled": [
                    "true"
                  ],
                  "usernameLDAPAttribute": [
                    "cn"
                  ],
                  "bindDn": [
                    "cn=YOUR_CN,ou=YOUR_SERVICE_ACCOUNT_OU,dc=YOUR_DOMAIN,dc=YOUR_LOCAL"
                  ],
                  "bindCredential": [
                    "$(env:LDAP_BIND_CREDS)"
                  ],
                  "changedSyncPeriod": [
                    "-1"
                  ],
                  "lastSync": [
                    "1622010092"
                  ],
                  "vendor": [
                    "ad"
                  ],
                  "uuidLDAPAttribute": [
                    "objectGUID"
                  ],
                  "connectionUrl": [
                    "ldaps://LDAPS.YOUR.DOMAIN"
                  ],
                  "allowKerberosAuthentication": [
                    "false"
                  ],
                  "syncRegistrations": [
                    "true"
                  ],
                  "authType": [
                    "simple"
                  ],
                  "debug": [
                    "false"
                  ],
                  "searchScope": [
                    "2"
                  ],
                  "useTruststoreSpi": [
                    "always"
                  ],
                  "priority": [
                    "0"
                  ],
                  "trustEmail": [
                    "false"
                  ],
                  "userObjectClasses": [
                    "person, organizationalPerson, user"
                  ],
                  "rdnLDAPAttribute": [
                    "cn"
                  ],
                  "editMode": [
                    "READ_ONLY"
                  ],
                  "validatePasswordPolicy": [
                    "false"
                  ],
                  "batchSizeForSync": [
                    "0"
                  ]
                }
              }
            ]
          },
          "clients": [
            {
              "clientId": "k8s-client",
              "name": "k8s-client",
              "description": "k8s-Client",
              "enabled": true,
              "protocol": "openid-connect",
              "clientAuthenticatorType": "client-secret",
              "secret": "$(env:CLIENT_SECRET)",
              "directAccessGrantsEnabled": true,
              "authorizationServicesEnabled": true,
              "serviceAccountsEnabled": true,
              "protocolMappers": [
                {
                  "name": "Client IP Address",
                  "protocol": "openid-connect",
                  "protocolMapper": "oidc-usersessionmodel-note-mapper",
                  "consentRequired": false,
                  "config": {
                    "user.session.note": "clientAddress",
                    "id.token.claim": "true",
                    "access.token.claim": "true",
                    "claim.name": "clientAddress",
                    "jsonType.label": "String"
                  }
                },
                {
                  "name": "Client ID",
                  "protocol": "openid-connect",
                  "protocolMapper": "oidc-usersessionmodel-note-mapper",
                  "consentRequired": false,
                  "config": {
                    "user.session.note": "clientId",
                    "id.token.claim": "true",
                    "access.token.claim": "true",
                    "claim.name": "clientId",
                    "jsonType.label": "String"
                  }
                },
                {
                  "name": "groups",
                  "protocol": "openid-connect",
                  "protocolMapper": "oidc-group-membership-mapper",
                  "consentRequired": false,
                  "config": {
                    "full.path": "true",
                    "id.token.claim": "true",
                    "access.token.claim": "true",
                    "claim.name": "groups",
                    "userinfo.token.claim": "true"
                  }
                },
                {
                  "name": "Client Host",
                  "protocol": "openid-connect",
                  "protocolMapper": "oidc-usersessionmodel-note-mapper",
                  "consentRequired": false,
                  "config": {
                    "user.session.note": "clientHost",
                    "id.token.claim": "true",
                    "access.token.claim": "true",
                    "claim.name": "clientHost",
                    "jsonType.label": "String"
                  }
                }
              ],
              "authorizationSettings": {
                "allowRemoteResourceManagement": true,
                "policyEnforcementMode": "ENFORCING",
                "resources": [
                  {
                    "name": "Default Resource",
                    "type": "urn:k8s-client:resources:default",
                    "ownerManagedAccess": false,
                    "attributes": {},
                    "uris": [
                      "/*"
                    ]
                  }
                ],
                "policies": [
                  {
                    "name": "Default Policy",
                    "description": "A policy that grants access only for users within this realm",
                    "type": "js",
                    "logic": "POSITIVE",
                    "decisionStrategy": "AFFIRMATIVE",
                    "config": {
                      "code": "// by default, grants any permission associated with this policy\n$evaluation.grant();\n"
                    }
                  },
                  {
                    "name": "Default Permission",
                    "description": "A permission that applies to the default resource type",
                    "type": "resource",
                    "logic": "POSITIVE",
                    "decisionStrategy": "UNANIMOUS",
                    "config": {
                      "defaultResourceType": "urn:k8s-client:resources:default",
                      "applyPolicies": "[\"Default Policy\"]"
                    }
                  }
                ],
                "scopes": [],
                "decisionStrategy": "UNANIMOUS"
              },
              "redirectUris": [
                "*"
              ],
              "webOrigins": [
                "*"
              ]
            }
          ]
        }
  ```
