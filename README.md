# Documentação para instalação do Vault.

- Neste procedimento foi utilizado o uma VM;

  - CentOS 7 (Vault 0.7.3), vault.example.com;
  - Caso esteja com o firewall habilitado não esquecer de realizar a liberação;
  - Não é preciso desabilitar o SELINUX;

- Por questões de segurança, criaremos um usuário para executar o serviço do vault.

#### 1. Criamos o usuário vault com o comando abaixo.
```shell
useradd -r -g daemon -d /usr/local/vault -m -s /sbin/nologin -c "Vault User" vault
```

#### 2. Ajustado permissionamento dos diretórios.
```shell
chown vault.daemon /usr/local/vault
chmod 700 /usr/local/vault
```

#### 3. Para realizar o download do binário precisamos ter alguns pacotes básicos.
```shell
yum install vim wget unzip jq
```

#### 4. Realizamos o download do binário e colocamos em um ambiente de bin para todos os usuários.
```shell
wget https://releases.hashicorp.com/vault/0.7.3/vault_0.7.3_linux_amd64.zip
unzip vault_0.7.3_linux_amd64.zip
mv vault /usr/bin/
```

#### 5. Podemos testar com o comando vault, o mesmo tem que retornar o help dos comandos.
```shell
vault 
usage: vault [-version] [-help] <command> [args]

Common commands:
    delete           Delete operation on secrets in Vault
    path-help        Look up the help for a path
    read             Read data or secrets from Vault
    renew            Renew the lease of a secret
...
```

#### 6. Agora iremos criar uma estrutura de diretório para o conf do servidor vault.
```shell
mkdir /etc/vault         
chown vault.root /etc/vault 
chmod 750 /etc/vault
```

#### 7. Após criar a estrutura, vamo criar o arquivo de configuração server.hcl, dentro diretório /etc/vault/, que irá configurar o bind em todas as portas e utilizará o diretório /usr/local/vault/data como armazenamento dos segredos.
```shell
listener "tcp" {
 address = "0.0.0.0:8200"
 tls_disable = 1
}
backend "file" {
 path = "/usr/local/vault/data"
}
disable_mlock = true
```

#### 8. Após criar a configuração do serviço, precisamos criar a unit de serviço, no arquivo /etc/systemd/system/vault.service
```shell
[Unit]
Description=Vault server daemon
After=network.target

[Service]
User=vault
ExecStart=/usr/bin/vault server -config=/etc/vault/server.hcl
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process

[Install]
WantedBy=multi-user.target
```
#### 9. Após criar a unit precisamos realizar o reload, start e enable da unit.
```shell
systemctl daemon-reload
systemctl start vault
systemctl enable vault
```

#### 10. Após o serviço iniciar, podemos iniciar a configuração do Vault, primeiramente precisamos iniciar o vault com o comando vault init. Porém primeiramente precisamos exportar uma variável para o endereço do Vault.
  - #### OBS: As keys geradas pelo vault init, são as chaves para o unseal do vault, caso essas keys sejam perdidas, extraviadas os segredos serão perdidos.
```shell
export VAULT_ADDR=http://vault.example.com:8200
vault init
```

#### 11. Após executar o comando, terá gerado um retorno na tela similar ao abaixo.
```shell
Unseal Key 1: <Valor da Key>
Unseal Key 2: <Valor da Key>
Unseal Key 3: <Valor da Key>
Unseal Key 4: <Valor da Key>
Unseal Key 5: <Valor da Key>
Initial Root Token: <Token inicial de autenticação>

Vault initialized with 5 keys and a key threshold of 3. Please
securely distribute the above keys. When the vault is re-sealed,
restarted, or stopped, you must provide at least 3 of these keys
to unseal it again.

Vault does not store the master key. Without at least 3 keys,
your vault will remain permanently sealed.
```

#### 12. Por padrão o vault precisa de três das cincos chaves geradas para realizar o unseal. Para realizar o unseal executamos o comando vault unseal. No nosso caso precisamos executar esse comando três vezes com keys diferentes, geradas no passo anterior.
```shell
vault unseal
Key (will be hidden):

Sealed: true
Key Shares: 5
Key Threshold: 3
Unseal Progress: 1
Unseal Nonce: 15210a3b-0ebc-b7e7-3014-13f29a7a38b9

vault unseal
Key (will be hidden): 

Sealed: true
Key Shares: 5
Key Threshold: 3
Unseal Progress: 2
Unseal Nonce: 15210a3b-0ebc-b7e7-3014-13f29a7a38b9

vault unseal
Key (will be hidden): 

Sealed: false
Key Shares: 5
Key Threshold: 3
Unseal Progress: 0
Unseal Nonce: 
```

#### 13. Apartir de agora o vault está disponível para acesso com o token de autenticação e disponível para acesso.
```shell
vault auth
Token (will be hidden): 

Successfully authenticated! You are now logged in.
token: d0151df4-8a35-583f-7f74-2d4ec08485c5
token_duration: 0
token_policies: [root]
```
- #### Documentações Utilizadas.

https://www.vaultproject.io/
