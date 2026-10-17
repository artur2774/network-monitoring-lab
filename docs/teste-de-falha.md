# Teste controlado de falha e recuperação

## Objetivo e escopo

Validar que o Zabbix detecta indisponibilidade do agente do Ubuntu, gera um evento e o resolve automaticamente após a recuperação. A VM, SSH, Apache, PostgreSQL e Zabbix Server permaneceram em execução.

## Procedimento realizado

1. Confirmar os dois hosts com ZBX verde e métricas recentes.
2. Parar o agente no Ubuntu pela sessão SSH:

```bash
sudo systemctl stop zabbix-agent2
```

3. Acompanhar Monitoring → Hosts e Monitoring → Problems.
4. Confirmar ZBX vermelho e aguardar a janela da trigger.
5. Registrar o problema Linux: Zabbix agent is not available (for 3m).
6. Iniciar o agente:

```bash
sudo systemctl start zabbix-agent2
```

7. Confirmar ZBX verde, retomada da coleta e evento RESOLVED.

## Critério de disparo

A trigger habilitada usa a disponibilidade interna do agente:

```text
max(/Zabbix server/zabbix[host,agent,available],{$AGENT.TIMEOUT})=0
```

A macro herdada {$AGENT.TIMEOUT} foi consultada e seu valor era 3m. O evento depende da disponibilidade registrada e das avaliações do Zabbix; não se deve interpretar esse valor como garantia de disparo exatamente três minutos após o comando stop.

## Resultado observado

| Verificação | Resultado |
|---|---|
| Agente interrompido | ZBX vermelho |
| Trigger de indisponibilidade | Evento com severidade Average |
| Início do problema na interface | 16/10/2026, 23:53:26 |
| Recuperação registrada | 16/10/2026, 23:58:26 |
| Duração do evento | 5 minutos |
| Após iniciar o agente | ZBX verde e RESOLVED |

Os horários se referem ao evento registrado no frontend, configurado para America/Sao_Paulo. A duração do evento não é o tempo total desde o comando de parada.

## Evidência e interpretação

O histórico do problema registra início, recuperação e duração. O dashboard exibe uma lacuna no gráfico de CPU do Ubuntu durante a ausência de coleta. Isso valida o ciclo detecção → registro → recuperação; não comprova envio de notificações externas.
