# Курсовая работа по итогам модуля "DevOps и системное администрирование"

Курсовая работа необходима для проверки практических навыков, полученных в ходе прохождения курса "DevOps и системное администрирование".

Мы создадим и настроим виртуальное рабочее место. Позже вы сможете использовать эту систему для выполнения домашних заданий по курсу

## Задание

1. Создайте виртуальную машину Linux.
2. Установите ufw и разрешите к этой машине сессии на порты 22 и 443, при этом трафик на интерфейсе localhost (lo) должен ходить свободно на все порты.
3. Установите hashicorp vault ([инструкция по ссылке](https://learn.hashicorp.com/tutorials/vault/getting-started-install?in=vault/getting-started#install-vault)).
4. Cоздайте центр сертификации по инструкции ([ссылка](https://learn.hashicorp.com/tutorials/vault/pki-engine?in=vault/secrets-management)) и выпустите сертификат для использования его в настройке веб-сервера nginx (срок жизни сертификата - месяц).
5. Установите корневой сертификат созданного центра сертификации в доверенные в хостовой системе.
6. Установите nginx.
7. По инструкции ([ссылка](https://nginx.org/en/docs/http/configuring_https_servers.html)) настройте nginx на https, используя ранее подготовленный сертификат:
  - можно использовать стандартную стартовую страницу nginx для демонстрации работы сервера;
  - можно использовать и другой html файл, сделанный вами;
8. Откройте в браузере на хосте https адрес страницы, которую обслуживает сервер nginx.
9. Создайте скрипт, который будет генерировать новый сертификат в vault:
  - генерируем новый сертификат так, чтобы не переписывать конфиг nginx;
  - перезапускаем nginx для применения нового сертификата.
10. Поместите скрипт в crontab, чтобы сертификат обновлялся какого-то числа каждого месяца в удобное для вас время.

## Результат

Результатом курсовой работы должны быть снимки экрана или текст:

### 1. Процесс установки и настройки ufw

UFW уже установлен в Ubuntu 20.04 server.

![img.png](img.png)

Установим правила UFW по умолчанию, запретим все входящие и разрешим все исходящие:

```sh
dmitry@ubuntu-server:~$ sudo ufw default deny incoming
dmitry@ubuntu-server:~$ sudo ufw default allow outgoing
```

Далее откроем 22 и 443 порты:

```shell
dmitry@ubuntu-server:~$ sudo ufw allow 22
dmitry@ubuntu-server:~$ sudo ufw allow 443
```
Включаем ufw: 
```shell
dmitry@ubuntu-server:~$ sudo ufw enable
```

Разрешим трафик для интерфейса lo на все порты: 

```shell
dmitry@ubuntu-server:~$ sudo ufw allow in on lo
```

Результат: 
```shell
dmitry@ubuntu-server:~$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
22                         ALLOW       Anywhere                  
443                        ALLOW       Anywhere                  
Anywhere on lo             ALLOW       Anywhere                  
22 (v6)                    ALLOW       Anywhere (v6)             
443 (v6)                   ALLOW       Anywhere (v6)             
Anywhere (v6) on lo        ALLOW       Anywhere (v6) 
```



### 2. Процесс установки и выпуска сертификата с помощью hashicorp vault

Устанавливаем vault:
```shell
dmitry@ubuntu-server:~$ curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
OK
dmitry@ubuntu-server:~$ sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"

dmitry@ubuntu-server:~$ sudo apt-get update && sudo apt-get install vault

```
Устанавливаем jq:

```shell
dmitry@ubuntu-server:~$ sudo apt-get install jq
```

Сконфигурируем Vault c помощью файла config.hcl: 

```shell
storage "raft" {
  path    = "./vault/data"
  node_id = "node1"
}

listener "tcp" {
  address     = "127.0.0.1:8200"
  tls_disable = "true"
}

api_addr = "http://127.0.0.1:8200"
cluster_addr = "https://127.0.0.1:8201"
ui = true
```

Запуск сервера Vault: 

```shell
dmitry@ubuntu-server:~$ mkdir -p ./vault/data
dmitry@ubuntu-server:~$ vault server -config=config.hcl
==> Vault server configuration:

             Api Address: http://127.0.0.1:8200
                     Cgo: disabled
         Cluster Address: https://127.0.0.1:8201
              Go Version: go1.17.5
              Listener 1: tcp (addr: "127.0.0.1:8200", cluster address: "127.0.0.1:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
               Log Level: info
                   Mlock: supported: true, enabled: true
           Recovery Mode: false
                 Storage: raft (HA available)
                 Version: Vault v1.9.2
             Version Sha: f4c6d873e2767c0d6853b5d9ffc77b0d297bfbdf

==> Vault server started! Log data will stream in below:
```

Запустим новый терминал и установим VAULT_ADDR и инициализируем хранилище:
```shell
dmitry@ubuntu-server:~$ export VAULT_ADDR='http://127.0.0.1:8200'
dmitry@ubuntu-server:~$ vault operator init

```
В результате получим:
```shell
Unseal Key 1: 4C1MmYfsqyDgPisKrNl00FUKDjZbI+2GfJl54h9JlLzB
Unseal Key 2: PCzq6lx0t2g0GZPn5/SPW+DMmxhl8QaAkYBboyIkZqt+
Unseal Key 3: sh/7gonWVbb1p6IZm9mqHUELNPVOfTvv8cY/Y6QyAPvZ
Unseal Key 4: lnV0KobS2twqaxPr4O7pvfCyll/Zezj+Ot+yFPdRNB6w
Unseal Key 5: RzV2xgTr3Ds7nWSnyUVzkyf3FmXWb+PSdDcJLEFQ8odr

Initial Root Token: s.AerNVclgYVtRyBpEx0bMPNNO
```
Распечатываем хранилище: 
```shell
dmitry@ubuntu-server:~$ vault operator unseal
Unseal Key (will be hidden): 
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    1/3
Unseal Nonce       920ef0ef-18cf-3051-e708-185bddd1867b
Version            1.9.2
Storage Type       raft
HA Enabled         true
dmitry@ubuntu-server:~$ vault operator unseal
Unseal Key (will be hidden): 
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    2/3
Unseal Nonce       920ef0ef-18cf-3051-e708-185bddd1867b
Version            1.9.2
Storage Type       raft
HA Enabled         true
dmitry@ubuntu-server:~$ vault operator unseal
Unseal Key (will be hidden): 
Key                     Value
---                     -----
Seal Type               shamir
Initialized             true
Sealed                  false
Total Shares            5
Threshold               3
Version                 1.9.2
Storage Type            raft
Cluster Name            vault-cluster-963518b2
Cluster ID              26321149-6ab3-8c9e-b558-88163695fbad
HA Enabled              true
HA Cluster              n/a
HA Mode                 standby
Active Node Address     <none>
Raft Committed Index    25
Raft Applied Index      25
dmitry@ubuntu-server:~$ 

```

Пройдем процедуре аутентифиуации: 

```shell
dmitry@ubuntu-server:~$ vault login s.AerNVclgYVtRyBpEx0bMPNNO
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                s.AerNVclgYVtRyBpEx0bMPNNO
token_accessor       3UVFvFu6kJhicCPPI6jkCDVX
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]

```

1. Cоздаем корневой CA:

```shell
dmitry@ubuntu-server:~$ vault secrets enable pki
Success! Enabled the pki secrets engine at: pki/
dmitry@ubuntu-server:~$ vault secrets tune -max-lease-ttl=87600h pki
Success! Tuned the secrets engine at: pki/
dmitry@ubuntu-server:~$ vault write -field=certificate pki/root/generate/internal \
>      common_name="example.com" \
>      ttl=87600h > CA_cert.crt

dmitry@ubuntu-server:~$ vault write pki/config/urls \
>      issuing_certificates="$VAULT_ADDR/v1/pki/ca" \
>      crl_distribution_points="$VAULT_ADDR/v1/pki/crl"
Success! Data written to: pki/config/urls

```
2. Создаем промежуточный CA:

```shell
dmitry@ubuntu-server:~$ vault secrets enable -path=pki_int pki
Success! Enabled the pki secrets engine at: pki_int/
dmitry@ubuntu-server:~$ vault secrets tune -max-lease-ttl=43800h pki_int
Success! Tuned the secrets engine at: pki_int/
dmitry@ubuntu-server:~$ vault write -format=json pki_int/intermediate/generate/internal \
>      common_name="example.com Intermediate Authority" \
>      | jq -r '.data.csr' > pki_intermediate.csr
dmitry@ubuntu-server:~$ vault write -format=json pki/root/sign-intermediate csr=@pki_intermediate.csr \
>      format=pem_bundle ttl="43800h" \
>      | jq -r '.data.certificate' > intermediate.cert.pem
dmitry@ubuntu-server:~$ vault write pki_int/intermediate/set-signed certificate=@intermediate.cert.pem
Success! Data written to: pki_int/intermediate/set-signed


```
3. Создаем роль 
```shell
dmitry@ubuntu-server:~$ vault write pki_int/roles/example-dot-com \
>      allowed_domains="example.com" \
>      allow_subdomains=true \
>      max_ttl="720h"
Success! Data written to: pki_int/roles/example-dot-com

```

4. Запросим сертификат для поддомена test.example.com и запишем его в виде json cert.json: 

```shell
dmitry@ubuntu-server:~$ vault write -format=json pki_int/issue/example-dot-com common_name="test.example.com" ttl="720h" > cert.json

```


### 3.Процесс установки и настройки сервера nginx

```shell
dmitry@ubuntu-server:~$ sudo apt install nginx

```

Настроим блок сервера для нашего домена test.example.com 

```shell
dmitry@ubuntu-server:~$ sudo mkdir -p /var/www/test.example.com/html
dmitry@ubuntu-server:~$ sudo chmod -R 755 /var/www/test.example.com/
dmitry@ubuntu-server:~$ /var/www/test.example.com/html/index.html
dmitry@ubuntu-server:~$ sudo nano /etc/nginx/sites-available/test.example.com

```

Подготовим файлы сертификатов для nginx и поместим в папку /etc/nginx/ssl:
```shell
dmitry@ubuntu-server:~$ sudo mkdir /etc/nginx/ssl


```


- Страница сервера nginx в браузере хоста не содержит предупреждений 
- Скрипт генерации нового сертификата работает (сертификат сервера ngnix должен быть "зеленым")
- Crontab работает (выберите число и время так, чтобы показать что crontab запускается и делает что надо)

## Как сдавать курсовую работу

Курсовую работу выполните в файле readme.md в github репозитории. В личном кабинете отправьте на проверку ссылку на .md-файл в вашем репозитории.

Также вы можете выполнить задание в [Google Docs](https://docs.google.com/document/u/0/?tgif=d) и отправить в личном кабинете на проверку ссылку на ваш документ.
Если необходимо прикрепить дополнительные ссылки, просто добавьте их в свой Google Docs.

Перед тем как выслать ссылку, убедитесь, что ее содержимое не является приватным (открыто на комментирование всем, у кого есть ссылка), иначе преподаватель не сможет проверить работу. 
Ссылка на инструкцию [Как предоставить доступ к файлам и папкам на Google Диске](https://support.google.com/docs/answer/2494822?hl=ru&co=GENIE.Platform%3DDesktop).
