# 📘 Semana 08 — Active Directory Básico

**Fase:** 1 — Fundamentos

---

## 🎯 O que estudei essa semana

### Estrutura do AD
- **Domínio** (`lab.local`): fronteira administrativa. **Floresta**: um ou mais domínios. **DC**: guarda o diretório e autentica
- **Objetos:** usuários, computadores e grupos. **OU:** pasta lógica que aceita GPO. Contêineres como `Users` e `Computers` **não** aceitam GPO ligada a eles
- O diretório é consultado por **LDAP** (389, ou 636 com TLS). A 3268 é o catálogo global

### Kerberos (porta 88)
- O usuário prova quem é ao KDC (roda no DC) e recebe um **TGT**
- Com o TGT, pede um **ticket de serviço** (TGS) para cada recurso acessado
- Depende de **relógios próximos**: por padrão a tolerância é de 5 minutos, e o cálculo usa UTC

### Autorização e GPO
- Cada usuário e grupo tem um **SID**. O acesso depende dos grupos do usuário, montados no token **no momento do logon**
- **GPO:** configuração aplicada em massa a computadores e usuários, ligada ao domínio ou a uma OU

---

## 🧪 Exercícios práticos no lab

### Exercício 1 — Explorar o domínio (DC01)

```powershell
Get-ADDomain
Get-ADComputer -Filter * | Select-Object Name, DNSHostName
Get-ADUser -Filter * | Select-Object Name, SamAccountName, Enabled
Get-ADGroupMember "Domain Admins"
```

**Resultado:**
- Computadores no domínio: `DC01` e `WIN11-CLIENT`. O Kali e o SIEM01 não aparecem porque nunca foram ingressados
- Domain Admins: um único membro, `CN=Administrador,CN=Users,DC=lab,DC=local`. É o estado esperado num lab novo

**Achado:** a conta embutida aparece como **`Administrador`**, em português. O 4625 da Semana 01 registrou `Administrator`, o texto digitado na tela de login. Os nomes de objetos embutidos mudam com o idioma da instalação, e uma regra de detecção que procure só `Administrator` pode não casar. A conta administradora embutida sempre tem SID terminado em **-500**, em qualquer idioma. Não confirmei o `SamAccountName` dela.

### Exercício 2 — Criar OU, usuário e grupo

```powershell
New-ADOrganizationalUnit -Name "LabUsuarios" -Path "DC=lab,DC=local"
New-ADUser -Name "Maria Teste" -SamAccountName "mteste" ... -Path "OU=LabUsuarios,DC=lab,DC=local"
New-ADGroup -Name "GrupoTeste" -GroupScope Global -Path "OU=LabUsuarios,DC=lab,DC=local"
Add-ADGroupMember -Identity "GrupoTeste" -Members "mteste"
Get-ADUser mteste -Properties MemberOf | Select-Object Name, MemberOf
```

**O que aconteceu:** a primeira consulta de `MemberOf` voltou **vazia** (`{}`). O `Get-ADGroup` mostrou que o grupo existia, e o `Get-ADGroupMember` mostrou `Maria Teste` dentro dele. Na consulta seguinte, o `MemberOf` já listava `CN=GrupoTeste,OU=LabUsuarios,DC=lab,DC=local`.

Não identifiquei a causa da primeira resposta vazia. As hipóteses eram que a adição ainda não tinha rodado quando consultei, ou que a consulta mostrou um estado antigo. O `MemberOf` também não lista o grupo principal (`Domain Users`).

**Lição:** ausência de erro visível não prova que a ação funcionou. A confirmação vem de consultar o estado depois.

### Exercício 3 — Logon com usuário do domínio (WIN11-CLIENT)

Entrei como `LAB\mteste`. O `whoami /groups` mostrou `GrupoTeste`: o grupo entrou no token no momento do logon, sem nenhuma configuração na máquina. Se o usuário tivesse sido adicionado ao grupo depois de logado, o grupo só apareceria após novo logon.

**No Wazuh (Discover), na ordem do tempo:**

| Horário | Evento | Nível |
|---|---|---|
| 16:30:08 | Logon Failure - Unknown user or bad password | 5 |
| 16:30:47 | Windows Workstation Logon Success | 3 |
| 16:35:56 | Logon Failure - Unknown user or bad password | 5 |
| 16:37:53 | Windows Workstation Logon Success | 3 |

**Leitura:** uma falha seguida de um sucesso, com intervalos de dezenas de segundos e `logonType 2` (teclado), é compatível com erro de digitação. O mesmo par seria suspeito com dezenas de falhas em segundos, origem pela rede (`logonType 3`) ou várias contas. Nenhuma regra padrão liga a falha ao sucesso seguinte, e é isso que a correlação da Fase 2 vai fazer.

**Detalhes do 4625:**
- `subStatus` `0xc000006a`, igual ao da Semana 01. Indica usuário existente com senha errada. Eu tinha previsto uma diferença, e errei. O código de usuário inexistente é outro, e não o testei
- O agente agora é o `002`, com o nome `WIN11-CLIENT`, e os alertas antigos ficam sob o `001`
- A validação da senha acontece no DC01, que **não tem agente do Wazuh**, então esse lado do evento não chega ao SIEM

### Exercício 4 — Kerberos com `klist`

Logado como `mteste`:

| Momento | Ticket | Início |
|---|---|---|
| 1º `klist` | krbtgt/LAB.LOCAL (TGT) | 12:33:25 |
| 1º `klist` | LDAP/DC01.lab.local | 12:33:25 |
| Depois de `dir \\DC01.lab.local\SYSVOL` | cifs/DC01.lab.local (novo) | 12:36:02 |

- Todos com AES-256. Validade de 10 horas (até 22:33:25) e renovação por 7 dias
- O ticket de serviço termina junto com o TGT
- O `Get-Date` (9/11 12:37:35) é coerente com a linha do tempo
- O ticket `LDAP` já existia antes do acesso ao SYSVOL, provavelmente por consultas do Windows ao diretório no logon. É suposição, a foto não prova a causa
- O flag `initial` só aparece no TGT

### Exercício 5 — GPO `AvisoLogon`

GPO ligada ao domínio, exibindo uma mensagem antes do logon.

**1. O `gpresult` do DC01 mostrou três GPOs:** `Default Domain Controllers Policy`, `Default Domain Policy` e `AvisoLogon`. No WIN11-CLIENT, só as duas últimas, o que confirma que o cliente não está na OU `Domain Controllers`.

**2. O primeiro `gpupdate` no WIN11-CLIENT falhou na política de computador por relógio dessincronizado.** A política de usuário passou. Depois de acertar o relógio, a GPO foi aplicada (7:09:39 PM).

**3. Mesmo aplicada, a GPO estava errada.** O registro do cliente mostrava `legalnoticecaption` com o texto do laboratório e `legalnoticetext` vazio. O `Get-GPRegistryValue` no DC01 confirmou: a GPO tinha **só um valor**. Os dois `Set-GPRegistryValue` saíram com o mesmo `-ValueName`, e o segundo sobrescreveu o primeiro.

**4. Meu erro na correção:** digitei `Get-GPRegistryValue` onde precisava de `Set-GPRegistryValue`. O `Get` só lê, e o PowerShell leu o `-Value` como uma segunda ocorrência de `-ValueName`. Nada foi alterado. Com o `Set`, a correção funcionou, e as duas frases apareceram na tela de login.

**Três níveis de verificação da GPO:**

| Nível | Como confirmei |
|---|---|
| Aplicada | `gpresult` com `AvisoLogon` |
| Gravada | `reg query` no cliente |
| Com efeito | As duas frases na tela, antes da senha |

Ela passou nos dois primeiros e falhou no terceiro até a correção.

---

## 🕐 Achado: relógio do DC01

| | DC01 | WIN11-CLIENT |
|---|---|---|
| Fonte de tempo | `Local CMOS Clock` (Stratum 1) | `DC01.lab.local` (Stratum 2) |
| Fuso | Pacífico | América do Sul |

- O DC01 usa o relógio da própria VM como fonte. Se ele derivar, o domínio inteiro deriva junto
- O fuso diferente só muda a **exibição** (o horário gravado no evento é UTC) e não causa o erro de Kerberos. O desvio real, em minutos, é o que causa
- O `ReferenceId` do cliente (`0xC0A8380A`) é o IP `192.168.56.10` em hexadecimal

**A confirmar:**
- Como o relógio foi acertado (`w32tm /resync`, `Set-Date` ou nova tentativa automática)
- O desvio medido com `w32tm /stripchart /computer:192.168.56.10 /samples:3 /dataonly`

---

## 🔎 Ângulo de blue team

Cada ação deixa evento no DC01. O DC01 não tem agente do Wazuh, então esses eventos só existem no Event Viewer dele:

| Ação | Evento (a confirmar) |
|---|---|
| Criar usuário | 4720 |
| Entrar em grupo | 4728 ou 4732 |
| Pedir TGT | 4768 |
| Pedir ticket de serviço | 4769 |
| Criar ou alterar objeto de GPO | 5137 e 5136 |

**O que confirmei:** achei um **4769** no DC01 (`11/11/2026 19:36:29`, `Audit Success`), então a auditoria de tickets de serviço está ligada. Não liguei esse evento ao `mteste`, e ele é de outro dia que o `klist` das fotos.

**A confirmar:** os eventos 4768, 5136 e 5137, e a política de auditoria com `auditpol /get /category:*`.

**Lacuna de visibilidade:** o servidor mais crítico do lab é o único sem monitoramento. Instalar o agente no DC01 é candidato para a Fase 2.

---

## 💡 Lições aprendidas

1. O AD é a base da autenticação corporativa, e o Kerberos depende de relógios próximos. Esse é o ponto onde a Semana 06 voltou a aparecer.
2. A autorização vem dos grupos, montados no token **no logon**.
3. Uma GPO tem níveis de verificação: aplicada, gravada e com efeito. Passar em um não prova os outros.
4. Um comando pode falhar silenciosamente no resultado: o `MemberOf` vazio e a GPO com um valor só passaram sem erro.
5. O verbo do PowerShell diz o que o comando faz: `Get` lê, `Set` grava, `New` cria, `Remove` apaga.
6. O ruído de fundo do Kerberos é grande: contas de máquina (nomes terminados em `$`) pedem tickets o tempo todo.
7. Nomes de objetos embutidos mudam com o idioma. Detecção por SID é mais robusta.

---

## ✅ Checklist da semana

- [x] Entendi domínio, floresta, DC, OU e grupos
- [x] Entendi o fluxo do Kerberos (TGT e tickets de serviço)
- [x] Criei OU, usuário e grupo pelo PowerShell
- [x] Vi o TGT e o ticket de serviço com `klist`
- [x] Apliquei uma GPO e confirmei os três níveis
- [x] Rodei a limpeza (a confirmação de que tudo sumiu está pendente)
- [ ] Conferi os eventos 4768, 5136 e 5137 no DC01
- [ ] Medi o desvio de relógio com `stripchart`
- [ ] Quiz de fixação (resultado: __/6)
- [x] Documentei aqui
