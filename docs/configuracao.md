# Configuração e operação

Registro da implantação realizada no laboratório. Os comandos abaixo documentam as etapas; não execute novamente a criação e a importação do banco em uma instalação já pronta.

## 1. Recursos e acesso

Ubuntu srv-lab em 10.10.20.10, com 2 vCPUs e 4 GB fixos de RAM no Hyper-V. O cliente WireGuard do PC utiliza o endpoint 192.168.0.32:51820 e AllowedIPs 10.10.20.10/32. No OPNsense, as regras do túnel permitem SSH e HTTP ao servidor.

Na primeira aba do PowerShell do PC, com WireGuard ativo:

```powershell
ssh artur@10.10.20.10
```

Na segunda aba:

```powershell
ssh -L 8080:127.0.0.1:8080 artur@10.10.20.10
```

Manter a segunda sessão aberta e acessar http://localhost:8080/zabbix/. O encaminhamento permite usar o frontend sem adicionar uma liberação direta da porta 8080 no firewall.

## 2. Pacotes

Após apt update e apt upgrade, foi instalado o repositório oficial da série 7.0 para Ubuntu 26.04:

```bash
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-5+ubuntu26.04_all.deb
sudo dpkg -i zabbix-release_7.0-5+ubuntu26.04_all.deb
sudo apt update
sudo apt install zabbix-server-pgsql zabbix-frontend-php php-pgsql zabbix-apache-conf zabbix-sql-scripts zabbix-agent2 postgresql apache2 -y
```

O apt-cache policy confirmou 7.0.31 como versão candidata do repositório oficial. Quatro atualizações do Ubuntu ficaram temporariamente pendentes por phasing, sem forçar sua aplicação.

## 3. Banco PostgreSQL

```bash
sudo -u postgres createuser --pwprompt zabbix
sudo -u postgres createdb -O zabbix zabbix
zcat /usr/share/zabbix-sql-scripts/postgresql/server.sql.gz | psql -h localhost -U zabbix -d zabbix
```

A senha é definida interativamente. Em /etc/zabbix/zabbix_server.conf, a diretiva deve estar ativa, sem #:

```ini
DBPassword=<SENHA_DO_BANCO>
```

O valor acima é um marcador, não uma senha para reutilizar. A senha do banco é independente da senha do usuário Admin da interface web.

## 4. Apache e frontend

Como o Nginx já utilizava TCP/80, foi configurado Listen 8080 em /etc/apache2/ports.conf e <VirtualHost *:8080> em /etc/apache2/sites-available/000-default.conf.

O pacote de integração instalou /etc/zabbix/apache.conf, mas não disponibilizou automaticamente o link em conf-available. Correção aplicada:

```bash
sudo ln -s /etc/zabbix/apache.conf /etc/apache2/conf-available/zabbix.conf
sudo a2enconf zabbix
sudo a2enmod proxy proxy_fcgi
sudo apache2ctl configtest
sudo systemctl restart apache2
sudo systemctl enable --now apache2 zabbix-server
sudo systemctl enable zabbix-agent2
sudo systemctl restart zabbix-server zabbix-agent2
```

O frontend abriu e os pré-requisitos foram aprovados. O assistente utilizou PostgreSQL, host localhost, porta padrão, banco zabbix, schema public e usuário zabbix. Nome da instalação: Network Lab. Fuso: America/Sao_Paulo. A senha inicial de Admin foi alterada.

## 5. Ubuntu monitorado

Host Zabbix server, interface 127.0.0.1:10050, habilitado, com Linux by Zabbix agent e Zabbix server health. Confirmados ZBX verde, dados recentes e gráfico de CPU.

## 6. Agente do OPNsense

O firewall foi atualizado de 26.7 para 26.7.7 para permitir a instalação do plugin os-zabbix7-agent. Configuração em Services → Zabbix Agent:

| Campo | Valor |
|---|---|
| Enabled | Marcado |
| Hostname | OPNsense-LAB |
| Listen Port | 10050 |
| Listen IPs | 10.10.20.1 |
| Source IP | Vazio |
| Zabbix Servers | 10.10.20.10 |
| Enable sudo root permissions | Desmarcado |

Regra na interface SERVIDORES/VLAN20, acima do bloqueio de redes privadas:

| Campo | Valor |
|---|---|
| Action / IP / protocolo | Pass / IPv4 / TCP |
| Origem / porta | 10.10.20.10 / any |
| Destino / porta | 10.10.20.1 / 10050 |
| Descrição | PERMITIR_ZABBIX_OPNSENSE |

Na interface utilizada, a porta foi informada com Single port or range. Teste executado no Ubuntu:

```bash
nc -vz -w 3 10.10.20.1 10050
```

Resultado: conexão succeeded. O teste confirma TCP; a validação posterior de ZBX verde e métricas confirmou a comunicação do protocolo Zabbix.

No Zabbix foi criado OPNsense-LAB com interface Agent 10.10.20.1:10050, monitorado pelo Server e template FreeBSD by Zabbix agent.

## 7. Dashboard

Network Lab — Monitoramento:

- Graph: CPU — Ubuntu; host Zabbix server; item CPU utilization.
- Graph: CPU — OPNsense (user/system); host OPNsense-LAB; itens CPU user time e CPU system time.
- Problems: Alertas do laboratório; hosts Zabbix server e OPNsense-LAB.

## 8. Verificações úteis

```bash
systemctl status zabbix-server zabbix-agent2 apache2 --no-pager
sudo tail -n 30 /var/log/zabbix/zabbix_server.log
sudo ss -ltnp 'sport = :10051'
```

Um serviço active (running) não basta para comprovar a coleta: conferir também conexão ao banco, disponibilidade dos agentes e atualização dos valores no frontend.
