# Problemas encontrados e soluções

| Sintoma | Diagnóstico | Correção aplicada | Validação |
|---|---|---|---|
| Ubuntu mostrava cerca de 518 MB de RAM | Memória dinâmica do Hyper-V | Configurar 4 GB fixos mantendo 2 vCPUs | free -h mostrou cerca de 3,8 GiB |
| PC sem acesso direto às VLANs | Redes do laboratório no Hyper-V | Reutilizar WireGuard do primeiro projeto e conectar por SSH | Login artur@10.10.20.10 |
| apache2.service não encontrado | Apache não instalado explicitamente | Instalar apache2 | Serviço disponível |
| Apache falhou ao iniciar | ss e journal mostraram Nginx na porta 80; Address already in use | Apache em 8080 e VirtualHost ajustado | Resposta HTTP via túnel |
| /zabbix retornava 404 | Configuração não habilitada no Apache | Link de /etc/zabbix/apache.conf e a2enconf zabbix | Alias reconhecido |
| Invalid command ProxyTimeout | Módulo de proxy ausente | a2enmod proxy proxy_fcgi | Syntax OK e frontend aberto |
| Zabbix server is not running | Log: fe_sendauth: no password supplied | Remover # de DBPassword e reiniciar zabbix-server | Aviso desapareceu e coleta iniciou |
| Plugin do OPNsense não aparecia em Services | Instalação recusada por versão desatualizada | Atualizar para 26.7.7 e instalar os-zabbix7-agent | Página do agente disponível |
| Opção Other ausente na porta da regra | Interface usa outro seletor | Single port or range: 10050 | nc succeeded |
| ZBX vermelho sem problema imediato | Trigger exige janela de indisponibilidade | Consultar trigger e macro 3m; acompanhar Problems | Evento gerado |
| Busca de trigger vazia | Filtro unavailable não correspondia ao nome | Pesquisar agent | Trigger de disponibilidade encontrada |

## Comandos usados no diagnóstico

```bash
free -h
sudo ss -ltnp 'sport = :80'
sudo apache2ctl configtest
sudo journalctl -u apache2 -n 30 --no-pager
sudo tail -n 60 /var/log/zabbix/zabbix_server.log
sudo ss -ltnp 'sport = :10051'
nc -vz -w 3 10.10.20.1 10050
```

O aviso Apache AH00558 sobre ServerName não era a causa da falha de inicialização. A causa confirmada era o conflito na porta 80.
