# 📘 Semana 07 — Rede pelo Terminal e Scripting Básico

**Fase:** 1 — Fundamentos

---

## 🎯 O que estudei essa semana

### Rede pelo terminal
- `ss -tulpn`: portas em escuta e o processo por trás de cada uma
- `curl`: `-k` (ignora certificado), `-I` (só cabeçalhos), `-s` (silencioso), `-o` (saída), `-w` (formato)

### Filtrar e contar com pipes
- `grep`, `grep -oE` (extrai só o trecho que casa), `awk`
- `sort | uniq -c | sort -rn`: conta ocorrências e ordena do maior para o menor

### Bash básico
- `for`, `if`, código de saída (`$?`: 0 é sucesso, outro valor é falha)
- `#!/bin/bash` na primeira linha e `chmod +x script.sh`

### Reforço
- Bandit níveis 16 a 18 concluídos (19 e 20 pendentes: o sistema estava fora do ar)

---

## 🧪 Exercícios práticos no lab

### Exercício 1 — Quem escuta o quê (SIEM01)

```bash
sudo ss -tulpn
sudo ss -tulpn | grep -E "1514|1515|9200|443"
```

**Linha de base do SIEM01:**

| Endereço | Porta | Protocolo | Processo | Função |
|---|---|---|---|---|
| `0.0.0.0` | 22 | TCP | sshd | Acesso remoto |
| `0.0.0.0` | 443 | TCP | node | Dashboard |
| `0.0.0.0` | 1514 | TCP | wazuh-remoted | Eventos dos agentes |
| `0.0.0.0` | 1515 | TCP | wazuh-authd | Registro de agentes |
| `0.0.0.0` | 55000 | TCP | python3 (`wazuh_apid.py`) | API do Wazuh |
| `127.0.0.1` | 9200, 9300 | TCP | java | Indexer (só local) |
| `127.0.0.1` | 323 | UDP | chronyd | Controle do serviço de hora |
| `127.0.0.53` | 53 | UDP e TCP | systemd-resolved | Resolvedor DNS local |

**Resultado:**
- **Exposto à rede do lab:** 22, 443, 1514, 1515 e 55000
- **Só local:** 9200, 9300, 323 e 53 (em `127.0.0.53`)

**Hipótese da Semana 03 confirmada:** o nmap mostrou a 9200 como `closed`, e o `ss` mostra o java escutando em `127.0.0.1:9200`. Quando o Kali bate nessa porta pela rede, nada atende naquele endereço, o SIEM01 responde com RST e o nmap marca `closed`. Aqui `closed` não é firewall, é serviço restrito à máquina.

**A porta 55000 (API do Wazuh):**
- Processo: `wazuh_apid.py` (PID 1654, PPID 1, usuário `wazuh`), com o Python do próprio framework em `/var/ossec/`
- Escuta em `0.0.0.0`, ou seja, em todas as interfaces
- O nmap padrão não a mostrou, porque só cobre as 1000 portas mais comuns
- O nmap direcionado (`nmap -p 55000 192.168.56.20`) mostrou `open`

**Scan completo (`nmap -p- 192.168.56.20`, as 65535 portas TCP):**

| Porta | Estado | Nome que o nmap deu |
|---|---|---|
| 22/tcp | open | ssh |
| 443/tcp | open | https |
| 1514/tcp | open | fujitsu-dtcns |
| 1515/tcp | open | ifor-protocol |
| 55000/tcp | open | unknown |

O resultado bate com as cinco portas expostas da linha de base, sem nenhuma porta extra. Os nomes `fujitsu-dtcns` e `ifor-protocol` são palpites de uma lista antiga do nmap. Na prática são as portas de eventos e de registro de agentes do Wazuh. O scan completo foi só de TCP: as portas UDP (53 e 323) não entram, e as duas escutam só em endereços locais.

**Comparação das visões:**

| Visão | O que mostra |
|---|---|
| `ss` (de dentro) | O que o servidor **abre** |
| nmap padrão | Só 22 e 443 |
| nmap `-p-` | O que um atacante veria **se soubesse procurar** |

**Correções do caminho:**
1. A porta que li como `5500` era a **55000**: faltou um zero na digitação.
2. O processo da porta 53 não era do Wazuh, e sim o `systemd-resolved`, o resolvedor DNS local do Ubuntu, que por escutar em `127.0.0.53` só atende a própria máquina.
3. A porta de controle do `chronyd` é a **323**.

**Por que importa para blue team:** o `ss` responde "o que este servidor expõe, e quem expõe?", e o nmap responde "o que o mundo enxerga?". Quando as duas listas divergem sem explicação, algo está errado. Uma porta nova em `0.0.0.0` fora desta tabela, principalmente com processo desconhecido, é um achado. A 55000 é uma entrada administrativa aberta em todas as interfaces, e numa rede real seria restrita por firewall aos IPs de quem administra.

### Exercício 2 — `curl` além do básico

```bash
curl -k -I https://192.168.56.20
curl -sk -o /dev/null -w "%{http_code}\n" https://192.168.56.20
```

**Resultado:** `302 Found`, com `location: /app/login?`. O dashboard redireciona para o login e apaga qualquer cookie de sessão antigo (`Max-Age=0`, expiração em 1970).

| Cabeçalho | O que diz |
|---|---|
| `Secure; HttpOnly` (no cookie) | Só trafega por HTTPS, e scripts do navegador não leem |
| `x-frame-options: sameorigin` | Impede embutir o dashboard em outro site (clickjacking) |
| `cache-control: no-store...` | O navegador não deve guardar a página |
| `osd-name: siem01` | Entrega o nome da máquina a qualquer um que pergunte |

**Uso em monitoramento:** o código HTTP indica rápido o estado do serviço. `302` é o normal para este dashboard sem login. Um `000` (curl sem conexão) deve indicar serviço parado, previsão que não testei com o dashboard fora do ar.

**Reconhecimento:** sem credencial, os cabeçalhos revelam o caminho do login e o nome da máquina.

### Exercício 3 — Contar tentativas por IP

```bash
sudo journalctl -u ssh --no-pager | grep "Failed password" | grep -oE "from [0-9.]+" | sort | uniq -c | sort -rn
```

**Resultado:** 13 falhas no histórico inteiro, mas só 6 hoje (`--since "today"`). Os 7 restantes eram de dias anteriores, e contar o log sem olhar o tempo mistura datas.

**Formato da linha:**
```
sshd-session[PID]: Failed password for invalid user <usuário> from <IP> port <porta> ssh2
```
- `invalid user` indica conta inexistente. O nome tentado fica registrado como digitado
- A porta de origem muda a cada conexão
- O processo se chama `sshd-session` em vez de `sshd` (OpenSSH recente). A conferir se os decoders do Wazuh reconhecem esse nome

### Exercício 4 — Primeiro script (`check-lab.sh`)

Script em bash que percorre os IPs do lab e testa cada um com `ping -c 1 -W 1`.

**Resultado:**
- Com as três VMs ligadas: `.10`, `.11` e `.20` aparecem como UP
- Com o SIEM01 desligado: `.20` aparece como DOWN, os outros continuam UP

**Como funciona:** o `if` usa o código de saída do `ping` (0 é sucesso, outro valor é falha), e a saída é descartada com `> /dev/null 2>&1`.

**Limitação:** DOWN significa "não respondeu ao ping", e não "desligada". Um firewall que descarta ICMP produz o mesmo resultado, como vi na Semana 04. Um teste de porta TCP verifica o serviço, e não só a máquina.

### Exercício 5 — Mini detector de força bruta (`detecta-ssh.sh`)

Script que extrai os IPs das falhas de SSH do journal, conta por IP e avisa quando passa do `LIMITE`.

**Resultado:**
- `LIMITE=3`: `ALERTA: 192.168.56.7 teve 13 falhas SSH`. A contagem subia a cada nova tentativa pelo Kali
- `LIMITE=20`: saída vazia, já que 13 não passa de 20

**Cuidado com o silêncio:** um script que não imprime nada pode significar "tudo normal" ou "o script quebrou". Aqui eu sabia que era o primeiro caso porque tinha visto o alerta com o limite menor.

**Limitações (pergunta do exercício):**
1. Soma datas diferentes: o 13 inclui falhas de dias anteriores, e só 6 eram de hoje (Exercício 3)
2. Nunca zera: depois de passar do limite, o IP fica em alerta para sempre
3. Não distingue ritmo: 3 falhas em segundos e 3 falhas em meses dão o mesmo resultado

**O que um detector de verdade faz:** conta falhas dentro de uma **janela de tempo** ("N falhas em X segundos"), roda continuamente e correlaciona com um sucesso posterior. A Fase 2 resolve isso com correlação por frequência no Wazuh.

---

## 💡 Lições aprendidas

1. **`ss` e nmap respondem perguntas diferentes.** Um mostra o que o servidor abre, o outro o que o mundo enxerga. Linha de base é a comparação entre os dois.
2. **Serviço funcionando é diferente de serviço alcançável pela rede.** O indexer funciona, mas só em `127.0.0.1`.
3. **Contar sem tempo engana.** O 13 contra 6 mostrou isso na prática.
4. **O código de saída é a base de todo script.** O `if` lê 0 ou não-0, e é assim que `ping`, `grep` e quase todo comando dizem se deu certo.
5. **`DOWN` não é "desligado".** O ping mede só uma resposta ICMP, e firewall muda o resultado.
6. **Silêncio de script é ambíguo.** Vale uma linha de "executado, nada encontrado".

**Erros no quiz (4/6), e o que revelam:**
- Marquei que o nmap não escaneia portas acima de 9000. O `nmap -p-` já tinha encontrado a 55000 aberta. O scan padrão esconde portas fora das 1000 mais comuns, e a 9200 aparece `closed` porque alguém respondeu "não há serviço aqui".
- Marquei que a limitação do `detecta-ssh.sh` era só funcionar com SSH. Isso é limitação de escopo, e a principal é de lógica: ele conta falhas acumuladas, sem considerar o tempo.

---

## 🔭 Ideias para depois

- Uma versão do `detecta-ssh.sh` com janela de tempo (`--since "10 minutes ago"`), para ver o alerta aparecer e sumir. Não testei.
- Um `check-portas.sh` que testa serviços TCP em vez de ping. Não testei.
- Rodar o `nmap -sU` nas portas UDP do SIEM01.

---

## ✅ Checklist da semana

- [x] Sei usar `ss -tulpn` e ler endereço, porta e processo
- [x] Sei usar `curl` com `-I`, `-s` e `-w`
- [x] Consegui contar ocorrências com `sort | uniq -c | sort -rn`
- [x] Escrevi e rodei os dois scripts
- [x] Fiz os 5 exercícios no lab
- [ ] Bandit níveis 16 a 20 (feito até o 18; 19 e 20 pendentes)
- [x] Quiz de fixação (resultado: 4/6)
- [x] Documentei aqui
