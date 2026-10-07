# 👓 Write-up: Timeout de DNS com duas causas raiz sobrepostas

Tipo: diagnóstico de falha de rede/serviço
Ambiente: lab próprio isolado (VirtualBox, rede Host-only 192.168.56.0/24)
Ferramentas: nslookup, PowerShell (módulo DnsServer), Wireshark, Test-NetConnection
Resultado: resolvido, com duas correções independentes

---

## 1. Sintoma

No WIN11-CLIENT, a consulta de DNS ao domínio do lab dava timeout:

nslookup lab.local
DNS request timed out. timeout was 2 seconds.
Server: Unknown
Address: 192.168.56.10

## 2. Cenário

| Sistema | Papel | IP |
|---|---|---|
| DC01 | Domain Controller e servidor DNS do domínio lab.local | 192.168.56.10 |
| WIN11-CLIENT | Cliente com o problema | 192.168.56.11 |
| Kali | Máquina de comparação | 192.168.56.7 |

## 3. Investigação (eliminação por camada)

| # | Teste | Resultado | O que conclui |
| 1 | Resolve-DnsName lab.local no próprio DC01 | Funcionou | O serviço DNS estável |
| 2 | ping e Test-Connection - Port 53 do WIN111-CLIENT para o DC01 | Ambos funcionaram | Rede Básica e porta TCP/53 estão ok |
| 3 | nslookup -vc lab.local 192.168.56.10 (força TCp) | Funcionou | O problema aparece no caminho UDP, que é o padrão do nslookup |
| 4 | Regras de firewall, escuta do serviço (Get-NetUDPEndpoint) e configuração do DNS Server | Tudo normal | Não é firewall nem serviço mal configurado |
| 5 | nslookup lab.local 192.168.56.10 a partir do Kali | Funcionou, sem timeout | O servidor responde corretamente e a outra máquina |
| 6 | Captura simultânea no Wireshark (DC01 e WIN11-CLIENT), filtro udp-port == 53 | Consulta saiu, chegou e foi respondida | A comunicação UDP funcionou, então o foco passa para o conteúdo da resposta |

Detalhe da captura: o primeiro pacote visto era uma consulta wpad.lab.local, que o Windows dispara sozinho (descoberta automárica de proxy). Era ruído de fundo, e não a consulta que eu tinha pedido. Repeti a captura esperando alguns segundos antes de rodar o nslookup para isolar a consulta real.

Achado da captura: a resposta para lab.local trazia dois endereços A:

lab.local: type A, addr 192.168.56.10
lab.local: type A, addr 10.0.3.15

## 4. Causa raiz #1: registro DNS órfão

O 10.0.3.15 é um IP da rede privada que o VirualBOx entrega quando o adptador NAT está ligado. Um Domain Controller registra dinamicamente os IPs de todas as suas placas na zona DNS. Quando usei o NAT temporariamente no DC01, esse IP foi registrado. Ao desligar o adaptador, o registro continuou na zona, apontando para uma rede que o lab não alacança.

Como confirmei:
Get-DnsServerResourceRecord -ZoneName "lob.local" -RRType A -Name "@" | Format-List
A saída mostou dois registros com HostName: @ e DistinguishedName diferentes: um DC=@,... e outro DC=dc01,..., este com o IP errado.

Correção: altereio RecordData do registro incorreto para 192.168.56.10. Como resultado, ficaram dois registros A idênticos, o que não quebra a resolução. Remover a duplicata é opcional, e o comando de remoção filtra por RecordData, então pode atingir os dois registros. Se isso acontecer, o Add-DnsServerResourceRecordA recria o registro.

Resultado: o timeout continuou. havia uma segunda causa.

## 5. Causa raiz #2: ausência de zona de pesquisa reversa

Antes de resolver a consulta pedida, o nslookup tenta descobrir o nome do servidor DNs por uma consulta reversa (PRT). Era por isso que a saída mostrava Server: Unknown.

Como confirmei:
Get-DnsServerZone | Where-Object {$_.IsReverseLookupZone -eq $true }
Só existem as zonas reversar genéricas (0.in-addr.arpa, 127.in-addr.arpa, 255.in-addr.arpa). Nenhum cobria 192.168.56.0/24.

Correção:
Add-DnServerPrimaryZone - NetworkID "192.168.56.0/24" -ReplicationScope "Forest"
Add-DnsServerResourceRecordPtr -ZoneName "56.168.192.in-addr.arpa" -Name "10" -PtrDomainName "DC01.lab.local"

Validação:
ipconfig /flushdns
nslookup lab.local
A resposta veio rápida, sem timeout, com o IP correto.

## 6. Linha do tempo resumida

1. Sintoma: timeout no nslookup do cliente.
2. Isolei o problema ap UDP/DNS e descartei serviço, firewall e rede básica.
3. A captura de pacotes mostrou que a resposta chegava, mas com IP extra.
4. Corrigi o registro órfão. O timeout persistiu.
5. Investiguei o comportamento do nslookup e achei a zona reversa ausente.
6. Criei a zona reversa e o registro PTR. Resolvido.

## 7. Ponto de atenção

- Correlação com o NAT: o IP órfão veio de uma configuração temporária que fiz semanas antes. Mudanças temporárias deixxam rastros que só aparecem depois.
- O dig do Kali deu timeout durante a investigação, enquanto o nslookup funcionava. Não investiguei a causa. Provavelmente tem a ver com o dig fazer consultas diferentes das do nslookup, mas é uma hipótese, e não uma conclusão.
- A causa #1 pode não ter sido a que impedia o nslookup. A zona reversa ausente explica o sintoma sozinha, e o registro duplicado é um problema real mas separado. Não testei isolar cada uma delas.

## 8. Lições

1. Um mesmo sintoma pode ter várias causas raiz sobreposta. Resolver uma nem sempre resolve o problema.
2. Forçar TCP (nslookup -vc) separa rapidamente problemas de UDP.
3. Capturar pacotes nas duas pontas mostra o que de fato trafega, sem depender de suposição.
4. O nslookup depende de zona reversa mesmo quando não se pede resolução reversa.
5. Ruído de fundo (com o wpad) pode ser confidido com o tráfego do teste. Conferir o timing da captura evita conclusões erradas.

## 9. Relevância para blue team

- Um registro DNS que aponta para um IP inesperado é também um indicador clássico em investigações: DNS adulterado pode redirecionar tráfeg. Aqui a causa foi benigna, mas o método de verificação (comparar a resposta DNS com o inventário real da rede) é o mesmo.
- Saber o que é tráfego normal do Windows (wpad, cosultas reversas) reduz falso positivo ao analizar capturas.