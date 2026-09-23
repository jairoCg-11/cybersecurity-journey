# Semana 03 - DNS, DHCP e Protocolos de Aplicação

Fase: 1 - Fundamentos

---

## O que estudei essa semana

### DNS (Domain Name Sistem)
- Função: traduzir nomes (lab.local) em endereços IP
- Hieraquia: root servers -> TLD -> domínio -> registro específico
- Tipos de registro: A (nome->IPv4), AAAA (nome-> IPv6), CNAME (apelido), PTR(IP->nome, reverso)
- No meu lab: o DC01 atua como servidor DNS autoritativo do domíno lab.local.

### DHCP (Dynamic Host Configuration Protocol)
- Função: distribuir IP, máscara, gateway, e DNS automaticamente
- Processo DORA: Discover -> Offer -> Request -> Acknowledge (broadcast em todas as etapas, já que a máquina ainda não tem IP definido)
- No meu lab: só o Kali usa DHCP (servido pelo próprio VirtualBox, 192.168.56.1); as outras 3 VM's usam IP fixo

### Protocolos de aplicação e porta

| Protocolo | Porta | Função |
|---|---|---|
| DNS | 53 | Resolução de nomes |
| Kerberos | 88 | Autenticação no domínio |
| LDAP | 389 | Consuta ao diretório (usuários, grupos, computadores) |
| LDAPS | 636 | LDAP com criptografia TLS |
| SMB | 445 | Compartilhamento de arquivos/impressorass, replicação de GPO |
| SSH | 22 | Acesso remoto seguro a terminal |
| HTTPS | 443 | Navegação web criptografada |
| WinRM | 5985 | Adminstração remota via PowerShell |

---

## Exercícios práticos no lab

### Exercício 3 - DHCP em ação (ciclo DORA completo)

No Kali, forcei a liberação do IP e um novo pedido do zero:
sudo dhclient -r eth0
sudo dhclient -v eth0

Resultado (ciclo completo capturado)
DHCPDISCOVER on eth0 to 255.255.255.255 port 67 interval 8
DHCPOFFER of 192.168.56.7 from 192.168.56.1
DHCPREQUEST for 192.168.56.7 on eth0 to 255.255.255.255 port 67
DHCPACK of 192.168.56.7 from 192.168.56.1
bound to 192.168.56.7 -- renewal in 1741 seconds.

Confirmado visualmente no Wireshark (filtro bootp) - os 4 pacotes do DORA apareceram na sequência esperada. Identifiquei que 192.168.56.1 (o prórpio VirtualBox) atua como servidor DHCP da rede Host-only.

Observação: Discover e Request são enviados via broadcast (255.255.255.255), pois a máquina ainda não possui IP configurado nesse momento - conecta com o conceito de Camada 2/3 estudado na Semana 01.

### Exercício 4 - Identificar protocolos por portas (nmap)

Scan do DC01:
nmap -sV 192.168.56.10
Porta relevante encontradas: 53 (DNS), 88 (Kerberos), 135 (MSRPC), 139 (NetBIOS), 389 (LDAP), 445 (SMB), 464 (kpasswd), 593/3268/3269 (RPC/LDAP variantes), 636 (LDAPS), 5985 (WinRM). Todas fazem sentido para o Domaind Controller.

Lição de segurança: esse conjunto de portas (88, 389, 445 especialmente) é a superfície de ataque mais visada em Domaind Controllerssss reais (ex: Kerberoasting via porta 88).

Scan do SIEM01:
nmap -sV 192.168.56.20
Resultado inicial mostrou só 22 (SSH) e 443 (HTTPS/dashboard) - bem menos do que o esperado, já que sabia que o Wazuh também usa 1514, 1515 e 9200.

Causa identificada: o nmap, por padrão, escaneia apenas as 1000 portas mais comuns - portas específicas de aplicação (como as do Wazuh) ficam de fora dessa lista padrão.

Confirmação com scan direcionado:
nmap -p 1514,1515,9200 192.198.56.20
Resultado: 1514/tcp open, 1515/tcpm open, 9200/tcp closed.

Lição de segurança: um scan "rápido" pode dar falsa sensação de que um servidor está mais fechado do que realmente está - sempre validar portas específicas de casa serviço namualmente. A porta 9200 aparecer fechada externamente (mesmo funcionando via curl no próprio SIEM01) sugere que o indexer só aceita conexões locais (127.0.0.1)- boa prática de segurança, reduzindo superfície de ataque exposta.

### Exercício 5 - Captura de consulta DNS ao vivo (tcpdump)

sudo tcpdump -i eth0 port 53 -n
Gerado com nslookup lab.local 192.168.56.10 no próprio Kali.

Resultado:
192.168.56.7.51043 > 192.168.56.10.53: 8182+ A? lab.local. (27)
192.168.56.10.53 > 192.168.56.7.51043: 8182+ 1/0/0 A 192.168.56.10 (43)
192.168.56.7.36835 > 192168.56.10.53: 3336+ AAAA? lab.local (27)
192.168.56.10.53 > 192.168.56.7.36835: 3336* 0/1/0 (79)

Observação: a consulta AAAA (IPv6) retornou vazia (sem registro), o que é esperado, pois o lab só usa IPv4. Confirmado que o DNS pe tráfego unicast (ponto a ponto) - diferente do DHCP, que é broadcast - por isso a captura só apareceu na interface do Kali, não nas demais VMs.

--- 

## Troubleshooting da semana - Timeout de DNS no WIN11-CLIENT

### Sintoma inicial
nslookup lab.local
DNS request timed out. timeout was 2 seconds. Server: Unknown, Address: 192.168.56.10

### Investigando (eliminação por camadas)

1. Testando localmente no DC01: Resolve-DnsName lab.local -> funcionou. Confirma que o seviço DNS em si estava saudável.
2. Testando ping e porta TCP/53 do WIN11-CLIENT: ambos funcionaram - descartou problema de rede básica ou firewall total.
3. Testando nslookup -vc (forcando TCP): funcionou perfeitamente, enquanto o nslookup padrão (UDP) travava - isolou o problema especificamente ao tráfico UDP.
4. Testando do Kali: nslookup lab.local 192.168.56.10 funcionou sem problemas - confirmou que o servidor respondia corretamente via UDP para outras máquinas, restando suspeita nas configurações específicas de DC01<->WIN11-CLIENT ou nos próprios registros DNS.
5. Captura com Wireshark simultâneo (DC01 + WIN11-CLIENT), filto udp.port == 53: revelou que a consulta saia do WIN11-CLIENT, chegava no DC01, e a resposta voltava - mas com dois endereços IP diferentes no mesmo registro:
    Answers:
        lab.local: type A, addr 192.168.56.10
        lab.local: type A, addr 10.0.3.15

### Causa raiz #1 - Registro DNS duplicado/incorreto
Um registro remanescente de uma configuração de rede NAT temporária (10.0.3.15) estava cadastrada na zona lab.local, fazendo o DC01 responder com dos IPs - um deles inacessível pela rede lab.

Correção:
Get-DnsServerResouceRecord -ZoneName "lab.local" - RRType A -Name "@" | Format-List
Identificado o registro incorreto(DistinguishedName diferente, mesmo HostName: @) e corrido o RecordData para 192.168.56.10

### Causa raiz #2 - Ausência de zona de pesquisa reversa
Mesmo após corrigir o registro A, o timeout persistiu. Verificação: Get-DnsZoneName | Where-Object { $_.isReverseLookupZone -eq $true }
Resultado: só existem zonas reversas genéricas (0.in-addr.arpa, 127.in-addr.arpa, 255.in-addr.arpa) - nenhuma cobrindo 192.168.56.0/24. O nslookup tentava resolver o PRT (nome reverso) do próprio servidor DNS antes de processar a consulta principal, e tratava nessa tentatica (por isso aparecia "Server: Unknown").

Correção:
Add-DnsServerPrimaryZone -NetworkID "192.168.56.0/24" -ReplicationScope "Forest"
Add-DnsServerResourceRecordPtr -ZoneName "56.168.192.in-addr.arpa" -Name "10" -PtrDomainName "DC01.lab.local"

### Resultado final
ipconfig /flushdns
nslookup lab.local
Resposta rápida, sem timeout, apenas como o IP correto.

### Lições aprendidas.
1. Um mesmo sintoma (timeout) pode ser múltiplas causas raiz sobrepostas- resolver uma nem sempre resolve o problema por completo.
2. Diferenciar TCP vs UDP no diagnóstico (nslookup -vc) é uma técnica poderosa para isolar rapidamente em qual camada/protocolo está o problema.
3. Captura de pacotes (Wireshark) simultãneo em ambas as pontas é a forma mais confiável de confirmar exatamente o que está sendo enviado/recebido, sem depender de suposições.
4. Zonas de pesquisa reversa (PRT) são frequentemente esqucidas nos labs, mas ferramentas como `nslookup` dependem delas internamente mesmo quando não solicitadas explicitamente.
5. Registros DNS obsoletos (de configurações de rede temporárias, com NAT) podem permanecer cadastradas e causas problemas muito depois da configuração orifinal ter sido desfeita.

---

## Checklist da semana

- [x] Entendi como o DNS resolve nomes em IPs, incluindo tipos de registro (A, AAAA, PTR)
- [x] Entendi o processo DORA do DHCP e identifiquei o servidor DHCP da rede (VirtualBox)
- [x] Memorizei portas padrão dos principais protocolos e testei com nmap
- [x] Aprendi a diferença entre scan padrão e scan completo de portas no nmap
- [x] Capturei tráfego DNS ao vivo com tcpdump
- [x] Resolvi troubleshooting real de DNS com múltiplas causas raiz (registro ducplicatdo + zona reversa ausente)
- [x] Documentei aqui