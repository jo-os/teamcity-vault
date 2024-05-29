# teamcity-vault
Teamcity + Vault

**Teamcity**
Ставим плагин - https://plugins.jetbrains.com/plugin/10011-hashicorp-vault-support

**Vault**
1. Aктивируем approle для подключения из teamcity - https://developer.hashicorp.com/vault/docs/auth/approle, https://habr.com/ru/articles/653927/
```
    vault auth enable -tls-skip-verify approle
```
2. Cоздаем правило app_policy_name на доступ только к my/testing/secret/*
```
    vault policy write -tls-skip-verify app_policy_name -<<EOF
    # Read-only permission on secrets stored at 'my/testing/secret/*'
    path "my/data/testing/secret/*" {
    capabilities = [ "read" ]
    }
    EOF
```
3. Cоздаем роль my-app-role с привязкой к правилу app_policy_name - https://developer.hashicorp.com/vault/api-docs/auth/approle#secret_id_ttl (при создании еще можно ограничить доступ по сети - secret_id_bound_cidrs="0.0.0.0/0","127.0.0.1/32", token_bound_cidrs="0.0.0.0/0","127.0.0.1/32", а так же количество использования Secret ID - secret_id_num_uses=100 )
```
    vault write -tls-skip-verify auth/approle/role/my-app-role \
    token_policies="app_policy_name" \
    token_ttl=1h \
    token_max_ttl=4h \
    secret_id_ttl=0 policies="app_policy_name"
```
4. Проверяем что роль создалась
```
    vault read -tls-skip-verify auth/approle/role/my-app-role
```
5. Получаем role-id (необходима в teamcity для подключени)
```
    vault read -tls-skip-verify auth/approle/role/my-app-role/role-id
```
6. Получаем secret-id (необходима в teamcity для подключени)
```
    vault write -tls-skip-verify -f auth/approle/role/my-app-role/secret-id
```
7. Заходим под новым пользователем и проверяем доступ к секретам
```
    vault write -tls-skip-verify auth/approle/login role_id="Role-id" secret_id="Secret-id"
    export APP_TOKEN=token (вывод token из предыдушей команды)
    VAULT_TOKEN=$APP_TOKEN vault kv get -tls-skip-verify my/testing/secret/pass/main-secret
```

**Teamcity**

1. Подготовка подключения https по self-signed сертификату
```
    Берем сертификат Vault - /opt/vault/tls/tls.crt
    Делаем подходящий для TeamCity PEM - openssl x509 -in tls.crt -out vault.pem -outform PEM
    Добавляем в Administration - Root project - SSL / HTTPS Certificates
    Теперь TeamCity будет доверять self-signed сертификату
```
2. Создаем Connection к Vault
```
    Создаем в Administration - Root project - Connections - Add Connection
    Connection Type: Haashicorp Vault
    Display name: HashiCorp-VaultTest
    Vault URL: https://vault.server:8200
    AppRole auth endpoint path: approle
    AppRole Role ID: Role-id (5. Получаем role-id)
    AppRole Secret ID: Secret-id (6. Получаем secret-id)
    Test Connection - Save
```
3. Создаем Parameter
```
    Создаем в Administration - Project - Parameters - Add new parameter
    Name: TestVault
    Edit ->
    Type: Remote
    Remote Connection Type: HashiCorp Vault Parameter
    Vault ID: HashiCorp-VaultTest
    Vault Query: my/testing/secret/pass/main-secret!/my-password
```

------------------------------------------------------------------

**Мониторинг Vault**

Для сбора метрик необходимо добавить в vault.hcl
```
    telemetry {
    disable_hostname = true
    prometheus_retention_time = "12h"
    }
```
Сразу работает на single node, но не будет работать в кластере
```
    The /v1/sys/metrics endpoint is only accessible on active nodes and automatically disabled on standby nodes. You can enable the /v1/sys/metrics endpoint on standby nodes by enabling unauthenticated metrics access. If unauthenticated metrics access is enabled, the standby node will respond with its own metrics. If unauthenticated metrics access is not enabled, then a standby node will attempt to service the request but fail and then redirect the request to the active node.
```
Можно добавить в vault.hcl (надо тестировать правильно ли будут собираться метрики)
```
    listener "tcp" {
    telemetry {
        unauthenticated_metrics_access = "true"
    }}
```
Проверка
```
curl -k --header "X-Vault-Token: TOKEN" 'https://127.0.0.1:8200/v1/sys/metrics?format=prometheus'
https://developer.hashicorp.com/vault/docs/configuration/telemetry#prometheus
https://developer.hashicorp.com/vault/docs/configuration/listener/tcp#telemetry-parameters
```


