# 📘 Semana  - Linux na Prática: Navegação, Permissões e Usuários

Fase: 1 - Fundamentos

---

## O que estudei essa semana

### Navegação e estrutura de diretórios
- Comandos: pwd, ls -la, cd, cat, less, find, grep
- Diretórios-chave: /etc (configurações), /var/log (logs), /home (usuários), /tmp
- Padrão por aplicação: tudo do Wazuh fica sob /var/ossec, com /etc para configurações e logs/ para logs

### Permições
- Formato -rwx-xr--: tipo, depois dono, grupo e outros (r=4, w=2, x=1)
- chmod 640 arquivo (numéric) e chmod u+x (simbólico); chown dono:grupo arquivo
- Permição excessiva em arquivo de configuração sensível é achado clássico de auditoria

### usuários e grupos
- /etc/passwd (formato nome:x:UID:GID:comentário:home:shell),/etc/group, id, sudo, useradd, userdel
- O x no /etc/passwd indica que o hash da senha fica no /etc/shadow
- UID 0 é o root. Conta nova com UID 0 é alerta vermelho.

---

## 🧪 Exercícios práticos no lab (SIEM01)

### Exercício 1 - Onde ficam configurações e logs Wazuh

cd /var/ossec %% ls -la
sudo find /var/ossec -name "ossec.conf"

Resultado:
- Configurações: /var/ossec/etc/ossec.conf
- Logs: /var/ossec/logs (ossec.log, alerts/alerts.json, active-responses.log)

Correção do meu erro inicial: respondi journalctl para configuração e /var/log/dpkg.log para logs. O journalctl é uma ferramenta para ler logs do systemd, e o dpkg.log registra instalações de pacotes, não é log do Wazuh.

### Exercício 2 - Ler permissões

ls -l /var/ossec/etc/ossec.conf

Resultado: dono root, grupo wazuh, leitura e escrita para os dois, ninguém executa (-rw-rw----, ou 660).

O que eu aprendi: "quem pode" são os membros do grupo wazuh, e não só um usuário com esse nome. O último bloco (---) significa que outros usuários não têm acesso nenhum, e é de propósito: o arquivo tem configuração sensível so SIEM.

### Exercício 3 - Usuários e grupos do Wazuh

grep wazuh /etc/passwd
id
getent group wazuh

Resultado:
- Contas de serviço próprias: wazuh, wazuh-indexer e wazuh-dashboard
- wazuh:x:104:107::/var/ossec/:/sbin/nologin: UID 104, grupo principal 1107 (wazuh), home em /var/ossec/, shell nologin
- wazuh:x:107: no /etc/group: a lista de membros dica vazia porque o grupo é o principal da conta wazuh, e esse vínculo aparece no /etc/passwd, não aqui
- Minha conta (UID 1000) é comum e cai em "outros" no ossec.conf, por isso preciso de sudo para tê-lo

Príncipio de menor privilégio: cada componente roda com um usuário próprio, sem login e sem ser root. Se um deles for explorado, o atacante ganha só as permissões daquela conta.

Verificação útil para detecção:
awk -F: '$3 == 0 {print $1}' /etc/passwd
Lista contas com UID 0. O resultado normal é só root.

### Exercício 4 - Criar e apagar um usuário (e achar o rastro)

sudo useradd -m teste
sudo passwd teste
sudo grep useradd /var/log/auth.log
sudo userdel -r teste

Resultado: o log resgistrou a criação do grupo (new group: name=teste, GID=1001) e do usuário (new user: name=teste, UID=1001 ...) com data e hora.

O que o rastro responde: quem executou (entrada do sudo com meu usuário), o quê (useradd -m teste) e quando. Em uma investigação real, a pergunta é se havia uma mudança planejada que justificasse a criação. Se não havia, é possível presistência.

### Exercício 5 - Login falho por SSH (o equivalente Linux do evento 4625)

Do Kali:
ssh usuario_inexistente@192.168.56.20
No SIEM01:
sudo grep "Failed password" /var/log/auth.log

Resultado: o log registrou data, hora e o IP de origem da tentativa.

Coparação com a Semana 01:
| | Windows (Wazuh) | Linux (SSH) |
|---|---|---|
| Evento | ID 4625 | Linha Failed password no auth.log |
| Origem | 127.0.0.1 (login local) | IP do Kali (pela rede) |
| Usuário inexistente | Mensagem genérica | Aparece explícito como invalid user |

o formato muda, mas as perguntas do analista são as mesmas: quem tentou, de onde, em qual conta e quantas vezes.

---

## Lições aprendidas

1. Todo serviço bem configurado roda com conta própria, sem login e sem root. Isso limita o estrago de uma exploração
2. Permissão 660 em arquivo de configuração sensível é esperado. 666 ou 644 ali merece investigação
3. Cria uma conta e fazer login falho deixam rastros com quem, o quê e quando. É essa matéria-prima que o SIEM transforma em alerta.
4. Um login falho vindo de outra máquina da rede pesa mais que um local. Várias linhas invalid user do mesmo IP em poucos segundos é a assinatura de força bruta, que vou simular na Fase 3.
5. No /etc/passwd, o shell nologin é o normal para conta de serviço. Uma conta de serviço com /bin/bash indica que alguém a preparou para login.

--- 

## ✅ Checklist da semana
 
- [x] Naveguei pela árvore de diretórios e sei onde ficam configs e logs
- [x] Sei ler -rwxr-xr-- e entendi chmod numérico e simbólico
- [x] Entendi /etc/passwd, grupos e o papel do UID 0
- [x] Fiz os 5 exercícios no lab
- [x] Bandit até o nível 10
- [x] Documentei aqui