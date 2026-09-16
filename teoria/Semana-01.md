# Semana 01 - Fundamentos (Modelo OSI)

Fase 1 - Fundamentos

--- 

## O que estudei essa semana

### Redes - Modelo OSI (7 camadas)

| Camada | Nome | Função | Exemplo no meu lab |
|---|---|---|---|
|1 | Física | Meio de transmissão (cabos. wifi, fibra) | Rede Virtual Host-Only (vboxnet0) | 
| 2 | Enlace | Endereços MAC, switches | arp -a no Kali mostrando MAC's das VM's |
| 3 | Rede | Endereçamento IP, roteamento | IP's fixos das VM's (192.168.56.0/24) |
| 4 | Transporte | TCP (confiável) vs UDP (rápida, sem garantia) | ss -tulpn mostrando portas TCP/UDP |
| 5 | Sessão | Controle e manutenção da "conversa" entre dispositivos | Autenticação Kerberos no domínio lab.local |
| 6 | Apresentação | Formatação/tradução de dados, criptografia | TLS/SSL na conexão HTTPS com o Wazuh |
| 7 | Aplicação | Interface com usuário, protocolos usados diretamente | HTTP/Sm SSH, DNS |

---

## Exercícios práticos no lab

Mapeei cada camada usando comandos reais nas minhas VM's:

Camada 2 (Enlace):
ping -c 2 192.168.56.10
ping -c c 192.168.56.20
arp -a
Resultado: MAC's das VM's apareceram na tabela ARP após o ping (antes do ping, a tabela estava vazia - só é populada quando há tentativa de cominicação).

Camada 3 (Rede):
ip a
ip route

ss -tulpn

Camada 7 (Aplicação):
curl -k https://192.168.52.20
Resultado: conexão TCP + handshake TLS + resposta HTTP completo ("Connetion ... left intact")

---

## Teste de detecção - Wazuh capturando evento real

O que e eu fiz: Simulei uma tentativa de login com senha errada no WIN11-CLIENT e verifiquei a captura no dashboard do Wazuh (Discover, filtro agent.name: WIN11-CLIENT)

Evento capturado:
 - Event ID Windows: 4526 (falha de logon)
 - rule.description: "Logon Failure - Unknown user or bad password
 - rule.level: 5
 - logonType: 2 (interativo, direto no teclado da máquina)
 - Mapeamento MITRE ATT&CK: Tactic Impact, Technique Account Access Removal (T1531)

 Conclusão: O Wazuh está corretamente capturando,  indexando e classificando eventos de segurança do Windows, incluindo mapeamento automárico pro framework MITRE ATT&CK.

 ## Troubleshooting da semana

 ### Problema: apagar índices de logs da Wazuh (limpeza para reiniciar testes do zero
 
 Objetivo: apagar todos os alertas/logs indexados no Wazuh sem reinstalar nada.

 Tentativa 1 = Dev Tools do Dashboard:
 POST /wazuh-alerts-*/_delete_by_query
 { "query": {"match_all": {}}}
 Resultado: 404 - Not Found (endpoint não estava acessível dessa forma no dashboard)

 Tentativa 2 - via terminal do SIEM01, com senha errada:
 curl -k -u admin:<senha> -X DELETE "https://localhost:9200/wazuh-alerts-*"
 Resultado: 401 Unauthirized - senha usada do usuário admin no dashboard, não do admin do indexador (são credências diferentes!).

 Tentativa 3 - testando GET simples antes de deletar
 curl -k -u 'admin:<senha> -X GET "https://localhost:9200/_cat/indices?v"
 Resultado: 405 Method Not Allowed numa tentativa, depois 401 novamente - inconsistência causada por estar testando com senha diferentes em sequência sem isolar a variável.

 Causa raiz identificada: a senha do usuário admin do indexer (porta 9200) é diferente da senha do admin do dashboard (interface web). Elas ficam ambas no mesmo arquivo, mas sem seções distintas.

 Solução que funcionou:
 # 1. Recuperar a senha correta do indexer
 sudo tar -xf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt -O

 # 2. Testar autenticação básica primeiro (isolando a varável senha)
 curl -k -u 'admin:<senha_do_indexer>' https://localhost:9200/

 # 3. Confirmado (retornou JSON com cluster_name: wazuh-cluster) -> agora apagar
 curl -k -u 'admin:<senha_do_indexer>' -X DELETE "https://localhost:9200/wazuh-alerts-*"

 # 4. Confirmar que apagou
  curl -k -u 'admin:<senha_do_indexer>' https://localhost:9200/_cat/indices?v
  Resultado: {"acknowledged":true} - índices apagados com sucesso.

  Detalhe técnico extra: a senha do indexer continha caracteres especiais (?), por isso foi necessário uar aspas simples ('admin:<senha>') ao redor das credenciais no curl, evitando que o terminal interpretasse os caracteres incorretamente.

  ### Lições aprendidas
  1. Sempre isolar variáveis ao debugar (testar autenticação simples antes de operações destruticas como DELETE).
  2. Wazuh/OpenSearch tem múltiplos usuários admin para diferentes componentes (indexer vs dashoboard) - não assumir que a senha é a mesma.
  3. Usar aspas simples em credenciais com caracteres especias no curl evita erros de parsing do shell.
  4. Apagar os índices de logs *não remove* o cadastro do agente no manager - por isso o campo agent.name continuou mostrando o hostname antigo (DESKTOP-QMKFAPG) mesmo após limpeza.
  Esse são dois registros independentes.

  --

  Checklist da semana

  - [x] Entendi as 7 camadas do modelo OSI e exemplos práticos de cada uma
  - [x] Fiz o exercício prático mapeando cada camada no lab
  - [x] Testei detecção real de evento de segurança (login falho) no Wazuh
  - [x] Aprendi a interpretar campos de um alerta do Wazuh (rule.description, rule.level, MITRE ATT&CK)
  - [x] Resolvi trouleshooting de autenticação na API do OpenSearch/Indexer
  - [x] Documentei aqui