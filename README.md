# Documentação: Arquitetura Distribuída Zabbix

![Zabbix Version](https://img.shields.io/badge/Zabbix-7.0%20LTS-blue)
![MariaDB Version](https://img.shields.io/badge/MariaDB-10.11-lightgrey)
![Grafana Version](https://img.shields.io/badge/Grafana-v13-orange)

## Sumário
- [1. Visão Geral do Projeto](#1-visão-geral-do-projeto)
  - [Topologia de Rede (Hyper-V) Utilizada](#topologia-de-rede-hyper-v-utilizada)
- [PARTE I: MIGRAÇÃO E ATUALIZAÇÃO DA MATRIZ](#parte-i-migração-e-atualização-da-matriz)
  - [Fase 1: Preparação do Banco de Dados](#fase-1-preparação-do-banco-de-dados-servidor-novo-banco)
  - [Fase 2: O Backup e Congelamento](#fase-2-o-backup-e-congelamento-servidor-antigo-legado)
  - [Fase 3: Importação dos Dados](#fase-3-importação-dos-dados-servidor-novo-banco)
  - [Fase 4: O Motor Zabbix 7.0](#fase-4-o-motor-zabbix-70-servidor-novo-aplicação)
  - [Fase 5: Interface Web do Zabbix](#fase-5-interface-web-do-zabbix-servidor-novo-aplicação)
  - [Fase 6: Grafana v13 (Instalação Limpa e Importação)](#fase-6-grafana-v13-instalação-limpa-e-importação)
  - [Fase 7: Segurança Pós-Migração e Virada de Chave](#fase-7-segurança-pós-migração-e-virada-de-chave)
- [PARTE II: IMPLANTAÇÃO PADRONIZADA DE ZABBIX PROXY NOS CLIENTES](#parte-ii-implantação-padronizada-de-zabbix-proxy-nos-clientes)
  - [Fase 1: Preparação do Ambiente e Sistema (No Cliente)](#fase-1-preparação-do-ambiente-e-sistema-no-cliente)
  - [Fase 2: Configuração da Camada de Dados Local (MariaDB)](#fase-2-configuração-da-camada-de-dados-local-mariadb)
  - [Fase 3: Instalação do Ecossistema Proxy 7.0](#fase-3-instalação-do-ecossistema-proxy-70)
  - [Fase 4: Segurança e Comunicação (PSK)](#fase-4-segurança-e-comunicação-psk)
  - [Fase 5: Validação e Logs de Troubleshooting](#fase-5-validação-e-logs-de-troubleshooting)
  - [Fase 6: Desenho de Fluxo da Infraestrutura](#fase-6-desenho-de-fluxo-da-infraestrutura)

---

## 1. Visão Geral do Projeto

Este documento registra a modernização integral do nosso ambiente de monitoramento e estabelece o padrão oficial de implantação nas pontas. O projeto foi dividido em duas grandes frentes:

1. **A Matriz:** Migramos a infraestrutura "All-in-One" legada para uma arquitetura distribuída. O Banco de Dados (Cérebro) foi isolado em um servidor dedicado, enquanto a Aplicação (Zabbix + Grafana) passou a rodar em outro. O objetivo foi garantir mais performance, escalabilidade e segurança.
2. **Os Clientes:** Estabelecemos a padronização do Zabbix Proxy atuando como um "coletor local". Ele é responsável por processar dados de dispositivos locais (SNMP, agentes, câmeras) de forma resiliente e enviá-los de forma criptografada para o nosso servidor central atualizado.

### Topologia de Rede (Hyper-V) Utilizada

* **Servidor Antigo (Legado/Desativado):** IP `<IP_SERVIDOR_LEGADO>`
* **Novo Servidor de Banco de Dados:** IP `<IP_NOVO_BANCO_DADOS>` (MariaDB 10.11.14 - Debian 12)
* **Novo Servidor de Aplicação:** IP `<IP_NOVO_APLICACAO>` (Zabbix Server 7.0 + Grafana v13 - Debian 12)

---

## PARTE I: MIGRAÇÃO E ATUALIZAÇÃO DA MATRIZ

### Fase 1: Preparação do Banco de Dados (Servidor Novo Banco)

Acessamos o servidor de banco de dados e realizamos a instalação limpa do ambiente:

```bash
apt update && apt install mariadb-server -y
systemctl enable mariadb
systemctl start mariadb
```

Para otimizar o banco para o alto volume de dados do Zabbix, editamos o arquivo de configuração principal:

```bash
nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

As linhas abaixo foram adicionadas/alteradas na sessão `[mysqld]`:

```ini
bind-address = 0.0.0.0
innodb_buffer_pool_size = 4G
max_allowed_packet = 128M
innodb_log_file_size = 512M
log_bin_trust_function_creators = 1
```

> **Atenção:** A permissão `log_bin_trust_function_creators` foi ativada temporariamente para que o Zabbix 7.0 conseguisse criar as novas funções e triggers de banco durante o upgrade.

Após salvar o arquivo, o serviço foi reiniciado:

```bash
systemctl restart mariadb
```

Para evitar que o MariaDB bloqueasse a importação de triggers antigas (erro de Definer), acessamos o console do banco e criamos a estrutura com os usuários locais e remotos necessários:

```bash
mysql -u root -p
```

```sql
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;

CREATE USER 'zabbix'@'<IP_NOVO_APLICACAO>' IDENTIFIED BY '<SENHA_BANCO_ZABBIX>';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'<IP_NOVO_APLICACAO>';

CREATE USER 'zabbix'@'localhost' IDENTIFIED BY '<SENHA_BANCO_ZABBIX>';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';

FLUSH PRIVILEGES;
EXIT;
```

---

### Fase 2: O Backup e Congelamento (Servidor Antigo Legado)

Iniciamos a janela de manutenção interrompendo a gravação de dados para não corromper o banco durante a cópia:

```bash
systemctl stop zabbix-server zabbix-agent apache2 grafana-server
```

Como o banco era extenso (~30GB), adotamos o uso do `tmux` (um multiplexador de terminal) para garantir que o backup continuasse rodando nos bastidores mesmo se a conexão SSH caísse:

```bash
apt install tmux -y
tmux new -s backup_zabbix
```

Dentro da sessão segura do `tmux`, disparamos a extração dos dados:

```bash
mysqldump -u zabbix -p --single-transaction --routines --triggers --databases zabbix > /tmp/zabbix_backup_6.4.sql
```

Para acompanhar o progresso real da extração, abrimos uma segunda aba SSH e utilizamos o comando `watch` para monitorar o tamanho do arquivo crescendo:

```bash
watch -n 5 ls -lh /tmp/zabbix_backup_6.4.sql
```

Ao final, validamos a integridade do arquivo gerado:

```bash
tail -n 5 /tmp/zabbix_backup_6.4.sql
```

O dump foi transferido via rede para o novo servidor de Banco:

```bash
scp /tmp/zabbix_backup_6.4.sql root@<IP_NOVO_BANCO_DADOS>:/tmp/
```

> **Nota de Documentação:** Não trouxemos o arquivo `grafana.db` propositalmente. Constatamos que migrações de versões muito antigas do Grafana para a v13 quebram a estrutura do SQLite. Optamos pela importação via JSON detalhada na Fase 6.

---

### Fase 3: Importação dos Dados (Servidor Novo Banco)

No servidor de banco novo, executamos a importação dos dados legados para a nova estrutura:

```bash
mysql -u zabbix -p zabbix < /tmp/zabbix_backup_6.4.sql
```

Acessamos o banco e verificamos se a volumetria de dados correspondia à realidade para validar a importação:

```bash
mysql -u root -p
```

```sql
USE zabbix;
SELECT count(*) FROM hosts;
EXIT;
```

---

### Fase 4: O Motor Zabbix 7.0 (Servidor Novo Aplicação)

Para evitar que o Debian 12 bloqueasse os dicionários de rede (MIBs), ativamos os repositórios proprietários:

```bash
nano /etc/apt/sources.list
```

Adicionamos `non-free contrib` ao final das linhas que começavam com `deb`. Em seguida:

```bash
apt update
```

Os pacotes oficiais do Zabbix 7.0 e os tradutores SNMP foram instalados utilizando a sequência:

```bash
wget [https://repo.zabbix.com/zabbix/7.0/debian/pool/main/z/zabbix-release/zabbix-release_latest+debian12_all.deb](https://repo.zabbix.com/zabbix/7.0/debian/pool/main/z/zabbix-release/zabbix-release_latest+debian12_all.deb)
dpkg -i zabbix-release_latest+debian12_all.deb
apt update
apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent snmp snmp-mibs-downloader sqlite3 -y
download-mibs
```

> **Aviso:** Por se tratar de um ambiente migrado, **NÃO** executamos o comando padrão da Zabbix de importar o `server.sql.gz`. O banco já possuía os nossos dados estruturais.

O arquivo de configuração foi ajustado:

```bash
nano /etc/zabbix/zabbix_server.conf
```

Configuramos as credenciais e aplicamos uma regra vital (`CacheSize`) para impedir que o serviço travasse por falta de memória ao tentar ler todos os hosts do nosso banco de 30GB:

```ini
DBHost=<IP_NOVO_BANCO_DADOS>
DBName=zabbix
DBUser=zabbix
DBPassword=<SENHA_BANCO_ZABBIX>
CacheSize=1G
```

Iniciamos o serviço principal e imediatamente monitoramos o log para acompanhar o Zabbix 7.0 atualizando as tabelas da versão 6.4 de forma autônoma:

```bash
systemctl start zabbix-server
tail -f /var/log/zabbix/zabbix_server.log
```

O processo foi considerado um sucesso ao atingir a linha: `Zabbix Server has completed the database upgrade.`

---

### Fase 5: Interface Web do Zabbix (Servidor Novo Aplicação)

Para garantir a performance da página web e alinhar os horários dos eventos, o arquivo do PHP foi modificado:

```bash
nano /etc/php/8.2/apache2/php.ini
```

As linhas abaixo foram ajustadas (garantindo a remoção do comentário no início delas):

```ini
max_input_time = 300
date.timezone = America/Recife
```

O Apache foi reiniciado para aplicar as mudanças:

```bash
systemctl enable apache2
systemctl restart apache2
```

Acessamos `http://<IP_NOVO_APLICACAO>/zabbix`. Durante o Setup Web, no campo Host do banco de dados, substituímos o `localhost` pelo IP correto do nosso banco: `<IP_NOVO_BANCO_DADOS>`. O login foi realizado com sucesso utilizando as credenciais originais da equipe.

---

### Fase 6: Grafana v13 (Instalação Limpa e Importação)

Devido a incompatibilidades de schema entre a versão legada e a versão 13, a implantação do Grafana foi executada do zero, importando os Dashboards de maneira limpa. Adicionamos o repositório oficial e realizamos a instalação:

```bash
apt install -y apt-transport-https software-properties-common wget gpg
mkdir -p /etc/apt/keyrings/
wget -q -O- [https://apt.grafana.com/gpg.key](https://apt.grafana.com/gpg.key) | gpg --dearmor | tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] [https://apt.grafana.com](https://apt.grafana.com) stable main" | tee -a /etc/apt/sources.list.d/grafana.list
apt update
apt install grafana -y
```

Instalamos o plugin do Zabbix e iniciamos o serviço:

```bash
grafana-cli plugins install alexanderzobnin-zabbix-app
systemctl enable grafana-server
systemctl start grafana-server
```

Acessamos `http://<IP_NOVO_APLICACAO>:3000` e definimos a nova senha de administrador. O plugin do Zabbix foi ativado no menu *Administration > Plugins*. Em *Data Sources*, adicionamos o Zabbix com os parâmetros:

* **URL:** `http://localhost/zabbix/api_jsonrpc.php`
* **Versão:** 7.0
* **Detalhes:** Inserimos um login com permissões administrativas no Zabbix.

**Para a migração dos gráficos:** No Grafana Legado, os painéis foram abertos e extraídos via *Share > Export > Save to file* (gerando os JSONs). No Grafana Novo, usamos a função *New > Import*, fizemos o upload dos arquivos JSON e apontamos para o Data Source recém-criado, resgatando 100% da inteligência visual sem arrastar erros de banco.

---

### Fase 7: Segurança Pós-Migração e Virada de Chave

Com o upgrade do Zabbix finalizado, revogamos a permissão de criação de rotinas no MariaDB por segurança:

```bash
nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

A linha foi alterada para `log_bin_trust_function_creators = 0`. O serviço foi reiniciado:

```bash
systemctl restart mariadb
```

O acesso ao IP antigo também foi derrubado do banco:

```bash
mysql -u root -p
```

```sql
DROP USER 'zabbix'@'<IP_SERVIDOR_LEGADO>';
FLUSH PRIVILEGES;
EXIT;
```

---

## PARTE II: IMPLANTAÇÃO PADRONIZADA DE ZABBIX PROXY NOS CLIENTES

### Fase 1: Preparação do Ambiente e Sistema (No Cliente)

Após o primeiro boot, realizamos a fixação do endereço IP e liberamos o acesso remoto. Para identificar o nome da interface de rede:

```bash
ip a
```

Editamos o arquivo de rede:

```bash
nano /etc/network/interfaces
```

Configuração de IP estático aplicada (ajustar conforme a rede do cliente):

```ini
allow-hotplug ens18
iface ens18 inet static
address <IP_ESTATICO_CLIENTE>
netmask 255.255.255.0
gateway <GATEWAY_CLIENTE>
dns-nameservers 8.8.8.8 1.1.1.1
```

Comando para aplicar a nova rede:

```bash
systemctl restart networking
```

Para garantir o sincronismo de horário com o host físico (Hyper-V):

```bash
apt update && apt install hyperv-daemons -y
```

---

### Fase 2: Configuração da Camada de Dados Local (MariaDB)

Instalamos o banco de dados que servirá como persistência e buffer no cliente:

```bash
apt install mariadb-server -y
systemctl enable mariadb
systemctl start mariadb
```

Acessamos o console (`mysql -u root -p`) e executamos a criação da base e do usuário local do proxy:

```sql
CREATE DATABASE zabbix_proxy CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY '<SENHA_BANCO_PROXY>';
GRANT ALL PRIVILEGES ON zabbix_proxy.* TO 'zabbix'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

---

### Fase 3: Instalação do Ecossistema Proxy 7.0

Para suportar os pacotes de tradução SNMP da Cisco/Mikrotik, adicionamos o suporte a pacotes não-livres editando as fontes:

```bash
nano /etc/apt/sources.list
```

A linha principal deve receber os complementos no final:

```text
deb [http://deb.debian.org/debian/](http://deb.debian.org/debian/) bookworm main contrib non-free non-free-firmware
```

Na sequência, instalamos os binários oficiais:

```bash
wget [https://repo.zabbix.com/zabbix/7.0/debian/pool/main/z/zabbix-release/zabbix-release_latest+debian12_all.deb](https://repo.zabbix.com/zabbix/7.0/debian/pool/main/z/zabbix-release/zabbix-release_latest+debian12_all.deb)
dpkg -i zabbix-release_latest+debian12_all.deb
apt update
apt install zabbix-proxy-mysql zabbix-sql-scripts snmp snmp-mibs-downloader -y
```

Injetamos o Schema da estrutura do banco. Comando corrigido para a versão 7.0 (sem compactação .gz):

```bash
cat /usr/share/zabbix-sql-scripts/mysql/proxy.sql | mysql -u zabbix -p zabbix_proxy
```

---

### Fase 4: Segurança e Comunicação (PSK)

Geramos a chave Hexadecimal para blindar a comunicação entre o Cliente e a Matriz:

```bash
mkdir -p /etc/zabbix/keys
openssl rand -hex 32 > /etc/zabbix/keys/zabbix_proxy.psk
chown -R zabbix:zabbix /etc/zabbix/keys
chmod 640 /etc/zabbix/keys/zabbix_proxy.psk
```

Para visualizar e copiar a chave para o cadastro na Matriz:

```bash
cat /etc/zabbix/keys/zabbix_proxy.psk
```

Acessamos o arquivo de configuração para parametrizar a comunicação:

```bash
nano /etc/zabbix/zabbix_proxy.conf
```

Certificamo-nos de que os parâmetros abaixo estivessem configurados individualmente:

```ini
ProxyMode=0
Server=<IP_NOVO_APLICACAO>
Hostname=PRX-NOME_CLIENTE
DBName=zabbix_proxy
DBUser=zabbix
DBPassword=<SENHA_BANCO_PROXY>
TLSConnect=psk
TLSAccept=psk
TLSPSKIdentity=PSK-NOME_CLIENTE
TLSPSKFile=/etc/zabbix/keys/zabbix_proxy.psk
```

> **Nota Técnica Inclusa:** O IP do Server foi atualizado para apontar corretamente para a nova aplicação em `<IP_NOVO_APLICACAO>`.

Ativamos o serviço:

```bash
systemctl enable zabbix-proxy
systemctl restart zabbix-proxy
```

---

### Fase 5: Validação e Logs de Troubleshooting

Para testar a comunicação com um agente local (ativo do cliente) a partir do Proxy:

```bash
apt install zabbix-get -y
zabbix_get -s IP_DO_HOST -k agent.ping
```

Para validar se o Proxy está operando corretamente, monitorando bloqueios de Firewall ou recusas do servidor Matriz:

```bash
tail -f /var/log/zabbix/zabbix_proxy.log
```

---

### Fase 6: Desenho de Fluxo da Infraestrutura

O fluxo de tráfego e as conexões da arquitetura consolidada funcionam da seguinte forma:

* **Infraestrutura Cliente A & B:** Possuem coletores locais `ZABBIX-PROXY` que agregam o monitoramento local (SNMP/AGENT) de switches, roteadores, NVRs, câmeras e servidores.
* **Comunicação de Borda:** Os proxies de ambos os clientes comunicam-se via VPN/Internet através da porta padrão **10051** com o firewall de borda (`FORTINET/BORDA`).
* **Segmentação Interna (Matriz):** O firewall direciona as requisições dos proxies externos para o **Servidor Zabbix 7.0 + Grafana v13** (IP: `<IP_NOVO_APLICACAO>`). Este servidor de aplicação realiza o tráfego interno do MariaDB exclusivamente pela porta **3306** em direção ao servidor isolado dedicado (**Servidor Banco de Dados MariaDB v10.11.14** - IP: `<IP_NOVO_BANCO_DADOS>`).

---

## Agradecimentos

* À equipe pela confiança e autonomia, em especial a Breno Fernandes(Jairon) pelo direcionamento, informações e apoio sem isso esse projeto não 'vingaria'.
* Às comunidades do **Zabbix**, **MariaDB** e **Grafana** por fornecerem soluções open-source de altíssimo nível que movem a observabilidade e o monitoramento avançado no mundo inteiro.
