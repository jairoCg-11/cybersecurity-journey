# Arquitetura do Lab

## Visão geral

Lab de blue team isolado, rodando em VirtualBox sobre um Hackintosh (também usado para trabalho — por isso o isolamento de rede é obrigatório).

## Rede

- Tipo: **Host-only Adapter** (`vboxnet0`)
- Faixa: `192.168.56.0/24`
- Regra de segurança: nenhuma VM usa "Bridged Adapter" — isso evitaria contato com a rede real de trabalho

## Máquinas virtuais

| VM | Função | IP | SO |
|---|---|---|---|
| DC01 | Domain Controller | 192.168.56.10 | Windows Server 2022 |
| WIN11-CLIENT | Cliente monitorado | 192.168.56.11 | Windows 11 Enterprise (Evaluation) |
| SIEM01 | SIEM (Wazuh) | 192.168.56.20 | Ubuntu Server 24.04 |

## Fluxo de dados

1. `WIN11-CLIENT` gera eventos do sistema (login, processos, rede) via **Sysmon**
2. O **agente Wazuh** instalado no cliente envia esses eventos para o `SIEM01`
3. O `SIEM01` (Wazuh) recebe, indexa e exibe os eventos no dashboard
4. Análise e criação de regras de detecção acontecem no dashboard do Wazuh (`https://192.168.56.20`)