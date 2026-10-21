# 📘 Semana 06 - Processos, Seviços e systemd

Fase: 1 - Fundamentos

---

## 🎯 O que estudei essa semana

### Processos
- Todo programa em execução é um processo,com PID e PPID (o precesso que o criou)
- Comandos: ps aux, ps -eo pid,ppid,user,cmd, pstree -p, top
- A relação pai e filho contra quem iniciou o quê. Um bash filho de um serviço de rede seroa sinal clássico de exploração

### Serviços e systemd
- O systemd (PID 1) inicia e supervisiona os serviços
- systemd status|start|stop|restart|enable|disable
- active dis se o serviço roda agora. enable diz se ele sobe sozinho no boot. São independentes
- Arquivos de unidade ficam em `/etc/systemd/system/` e `/lib/sysemd/system/`

### Logs do systemd
- journalctl -u serviço, --since "30 min ago", -p warning, --no-pager

## Reforço
- Bandit níveis 11 a 15 concluídos
- LabEx: módulo "Become a Junior System Administrator" (dia 3 concluído)

--- 

## 🧪 Exercícios práticos no lab (SIEM01)

### Exercício 1 - Serviços o Wazuh

systemctl status wazuh-indexer
systemctl status wazuh-dashboard
systemctl status wazuh-manager

| Serviço | Main PID |
|---|---|
| wazuh-dashboard | 1131 |
| wazuh-indexer | 1395 |
| wazuh-manager | 1396 |

Os trẽs estavam active (running) e enabled, e por isso o SIEM01 volta com o Wazuh funcionando sozinho depois de ligar a VM. PID não é fixo: muda a cada boot, então um serviço se indentifica pelo nome e pelo caminho do execultável.

### Exercício 2 - Processos e árvore de pai e filho

ps aux | grep wazuh
pstree -p | grep -i wazuh
ps -eo pi,ppid,user,cmd | grep wazuh

**Resultado**:
- Dashboard e indexer rodam com contas próprias (`wazuh-dashboard` e `wazuh-indexer`, que o `ps` mostra cortadas com `+`)
- Os daemons do manager se dividem entre wazuh (db, analysisd, remoted, monitord) e root (authd, execd, syscheckd, logcollector, modulesd)
- PPID 1 nos daemons: quem os iniciou foi systemd
- Os python3 do framework são filhos da API (wazuh_apid.py, PID 1656)
- O grep que eu rodei tem com pai e minha sessão de terminal, o contraste entre algo que eu exectei e algo que o sistema iniciou
- Os nomes entre chaves no pstree, como {wazuh-remoted}, são threads, e não processos separados

Correção do meu erro: achei que o wazhu-monitord rodava como root. A captura mostrou wazuh.

### Exercício 3 - Log de um serviço

sudo journalctl -u wazuh-manager --since "30 min ago"

**Resultado**: o manager subiu às 11:44:10 (Starting Wazuh v4.14.7) e concluiu às 11:44:16 (Started wazuh-manager.service), cerca de 6 segundos. O log mostra uma linha Started por daemon, e a ordem bate com a ordem dos PIDs do Exercício 2.

Padrão normal de subida: Starting, sequência de Started, Completed. As linhas Process XXXX not used by Wazuh, removing... parecem limpeza de arquivos de PID da execução anterior. Essa leitura vem do texto, e eu não abri  os arquivos.

O que fugiria da linha de base: ERROR, Failed ou exited, reinícios repetidos, Stopping fora de manutenção, buracos no tempo e daemon faltando na sequência. Para filtrar:
sudo journalctl -u wazuh-manager -p warning --no-pager.

### Exercício 4 - Parar o dashboard e achar o rastro

sudo systemctl stop wazuh-dashboard
sudo journalctl -u wazuh-dashboard -n 20 --no-pager
sudo systemctl start wazuh-dashboard

**Resultado**: o journal do serviço registrou a data, hora e o PID da parada, e o navegador mostrou o dashboard offline. O manager continuou coletando, só a interface saiu do ar.

Quem fez: o registro do serviço não diz quem pediu a parada. Isso estava no registro do sudo, no journal (o auth.log não existia neste Ubuntu):

| Campo | Valor |
|---|---|
| Usuário | owl |
| Terminal | /dev/tty1 (console local) |
| Diretório | /home/owl |
| Executou com | root |
| Comando | /usr/bin/systemctl stop wazuh-dashboard |

tty1 é console local. Um pts/N costuma indicar sessão remota ou terminal gráfico.

---

## 🔎 Achado principal: o Wazuh alertou, mas como sudo genérico

A parada do dashboard gerou este alerta:

| Campo | Valor |
|---|---|
| rule.description | Successful sudo to ROOT Caching |
| rule.id | 5402 |
| rule.level | 3 |
| rule.mitre.id / técnica | T1548.003 (Sudo an Sudo Caching) |
| rule.mitre.tactic | Privilege Escalation, Defence Evasion |
| Location | journald |
| agent.id / agent.name | 000 / sem01 (o prórpio manager) |
| data.command | /usr/bin/systemctl stop wazuh-dashboard |

Leituras:
1. O alerta é de sudo, e não de serviço parado. Qualquer sudo recebe a mesma regra e o mesmo nível 3. Parar o monitoramento vale menos que uma falha de logon (nível 5). Isso é a lacuna a fechar na Fase 2, com uma regra própria para systemctl stop wazuh-*.
2. O Wazuh lê o journal do systemd. Eu tinha levantado a hipótese de que ele só lia arquivos, e o campo location: journald mostra o contrário.
3. O rótulo MITRE vem da regra, e não da ação. O uso foi legítimo, e o mapeamento só mostra que o Wazuh reconhece o sudo como ação de risco.

### Exercício 5 - Serviço de teste e persistência

Criei um serviço teste (ExecStart=bin/sleep 600), iniciei, habilitei com enable e depois limpei tudo. Reconstruí pelo journal quem criou, o quê e quando.

Persistência: foi um arquivo de seis linhas, um daemon-reload e um enable, tudo com sudo. Pouco trabalho para sobreviver a um reboot. Os rastros ficam em três lugares: o arquivo em /etc/systemd/system/, o registro do sudo e o alerta do Wazuh. Para listar o que está habilitado: systemctl list-unit-files --state-enabled. O analista procura unidades novas ou que apontem para executáveis em ligares estanhos (/tmp, /home, /dev/shm).

---

## ⏲️ Problema de relógio no SIEM01

Durante a semana percebi que o horário do SIEM01 não batia com o do meu computador.

| Observação | Valor |
|---|---|
| Alerta: full_log vs timestamp do dashboard | 12:02:02 vs 09:02:03, 3 horas de diferença |
| SIEM01 vc meu PC (primeira checagem) | 12:22 vs 10:56, 1h26 de diferença |
| timedatectl | System clock synchronized: no |
| Depois de ligar o NTP (com NAT temporário) | synchronized: yes; 14:113 vs 11:13 , 3h extras |

Interpretação: os 1h26 não se explica por fuso, e sim pelo relógio sem sincronização.
Depois de sincronizar, restaram 3h exatas, compatíveis com o servidor em UTC e o meu computador em UTC-3. Falta confirmar a linha Time zone do timedatectl

Efeito: os eventos gravados antes da correção (inclusive a parada do dashboard) ficam datados com o relógio defasado, e o Wazuh não reescreve o passado.

Por que importa: o relógio é a base de qualquer linha do tempo. Cuzar logs de máquinas com horários diferentes dá conclusões erradas. No Active Directory é pior: O Kerberos rejeita autenticação quando a diferença de horas entre as máquinass passa do limite configurado.

A conferir Get-Date no DC01 e no WIN-CLIENT

---

## 💡 Lições aprendidas

1. active e enable são coisas diferentes. O segundo é o que faz um serviço sobreviver ao reboot. e por isso é alvo de persistência.
2. Pai e filho contam história. Serviços têm PPID 1, Um terminal filho de serviço de rede seria suspeito.
3. Parar o monitoramento deia três rastros: o log do serviço (quando), o sudo (quem e de onde) e o SIEM (que, aqui, só viu um sudo genérico).
4. A detecção padrão tem lacunas. Nível 3 para parar o SIEM é pouco, e isso vira uma regra customizada na Fase 2.
5. Relógio errado invalida a linha do tempo. Antes de cruzar horários, verificar e sincronizar.
6. Linha de base primeiro. Saber como é uma subida normal é o que permite reconhecer a anormal.

---

## ✅ Checklist da semana

- [x] Sei listar processos e ler PID, PPID e usuários
- [x] Sei usar systemctl (status, start,stop, enable)
- [x] Sei ler logs de um serviço com journalctl
- [x] Entendi por que enable num serviço desconhecido -e suspeito
- [x] Fiz os 5 exercícios no lab
- [x] Bandit níveis 11 a 15
- [x] Módulo de systemd no LabEx
- [x] Quiz de fixação (resultado: 5/5)
- [x] Documentei aqui