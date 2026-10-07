# 👓 Write-up: Login falho no Windows (4625) vs Linux (SSH)

Tipo: investigação de alerta de autenticação
Ambiente: lab próprio isolado (VirtualBox, rede Host-only 192.168.56.0/24)
Ferramentas: Wazuh (dashboard/Discover), Sysmon, auth.log, Kali Linux

---

## 1. Objetivo

Gerar tentativas de login com falha em dois sistemas diferntes e comparar como casa um registra o evento, para treinar a leitura de log de autenticação, que é uma das tarefas mais comuns de um analista SOC N1.

## 2. Cenário

| Sistema | Papel | IP |
|---|---|---|
| WIN11-CLIENT | Cliente Windows 11, no domínio lab.local, monitorado por Sysmon + agente wazuh
| 192.168.56.11 |
| SIEM01 | Ubuntu Server 24.04 com Wazuh | 192.168.56.20 |
| Kali | Origem de tentativas por SSH | 192.168.56.7 |

## 3. Cassuo A: falha de logon no Windows

Ação: no WIN11-CLIENT, tentei locar como Adminstrador com senha errada, diretamente no teclado.

Detecção: o evento apareceu no Wazuh (Discover, filtro data.win.system.ID: "4625").

Campos principais do alerta:

| Campo | Valor | Interpretação |
|---|---|---|
| data.win.system.ID | 4625 | Falha de logon |
| rule.description | Logon Failure - Unknown user or bad password | Resumo da regra do Wazuh |
| rule.level | 5 | Severidade baixa a média (escala de 0 a 15) |
| rule.id | 60122 | Regr. que disparou |
| data.win.eventdata.targetUserName | Administrador | Conta-alvo | 
| data.win.eventdata.loginType | 2 | Interativo, direto na máquina |
| data.win.eventdata.ipAddress | 127.0.0.1 | Origem local |
| data.win.eventdata.status / subStatus | 0xc000006d / 0xc000006a | Usuário ou senha incorretos |
| rule.mitre.id | T1531 | Account Acess Removel (tática Impact) |

Leitura do analista: origem local, logon interativo e um único usuário. Isso é compatível com erro de digitação, e não com ataque, Um padrão preocupante teria logonType 3 ou 10 (rede ou RDP), IP de origem externo e várias contas diferentes em sequência.

Contexto do contador: o campo rule.firedtimes mostra quantas vezes aqulela regra disparou. Ele não zerou depois que apaquei os índices do Wazuh, o que indica que o contador fica o manager, separado do índice de logs.

##. Caso B: falha de login por SSH no Linux

Ação: do Kali, tentei SSH no SIEM com um usuário que não existe e senha errada, duas vezes:
ssh usuario@inexistente@192.168.56.220

Detecção: no SIEM01, busquei no log de autenticação:
sudo grep "Failed password" /var/log/auth.log e journalctl | grep "Failed password"

O que o log registrou:
data - hora - siem01 sshd-session[4597]: Failed password for invalid user usuario_inexistente from 192.168.56.7, port 34670 ssh2
data e hora, o IP de origem (o do Kali) e o nome de usuário tentado.
Para usuário inexistente, o sshd marca explicitamente como invalid user.

Leitura do analista: a origem é outra máquina da rede, e não um login local. Um nome de usuário que não existe sugere tentativa de adivinhar contas. Com uma ou duas tentativas isso é pouco relevante, mas dezenas de linha invalid user do mesmo IP em poucos segundos é a assinatura de força bruta.

## 5. Comparação

| | Windows | Linux |
|---|---|---|
| Fonte do log | Security log, coletado pelo agente Wazuh | /var/log/auth.log |
| Identificador | Event ID 4625 | Mensagem Failed password | 
| Origem da tentativa | 127.0.0.1 (local) | IP do Kali (rede) |
| Tipo de acesso | logonType: 2 (teclado) | SSH (remoto) |
| Usuário inexistente | Mensagem genérica de usuário ou senha ruim | Explícito como invalid user |
| Mapeamento MITRE no Wazuh | T1531 (já mapeado) | Não verificado nesse teste |

O formato muda, mas as preguntas são as mesmas: quem tentou, de onde, em qual conta e quantas vezes.

# 6. O que um SIEM faria com isso

- Uma falha isolada é ruim. Não jsutifica alerta.
- Muitas falhas do mesmo IP em pouco tempo justificam um alerta de força bruta.
- Várias falhas seguidas de um sucesso é o padrão mais importante: sugere que a senha foi adivinhada.
- Falha em conta que não existe reforça a hipótese de enumeração de usuários.

## 7. Limitações deste teste

- Gerei só uma ou duas tentativas por sistema, então não há volume suficiente para ver regras de correlação disparados.
- Não verifiquei se o Wazuh coletou o auth.log do SIEM01 como alerta. Isso fica como próximo passo.
- O Teste foi manual. Na faze 3, vou repetir com uma ferramenta de força bruta a partir do Kali e observar como a detecção muda com volume.

## 8. Lições

1. Os dois sistemas registram a mesma ação com formatos diferentes. Saber ler ambos é pré-requisito para triagem de alertas.
2. O tipo de logon (Windows) e a origem IP (Linux) separam um erro de digitação de uma tentativa de ataque.
3. Um log só tem valor se é coletado. Conferir se casa fonte está chegando ao SIEM é parte do trabalho de detecção.

## 9. Próximos passos

- Verificar se o auth.log do SIEM01 chega ao Wazuh como alerta.
- Repetir o teste com volume (Fase 3) e observar a correlação por frequência.
- Escrever uma regra customizada que alerte em múltiplas falhas seguidas do mesmo IP (Fase 2).