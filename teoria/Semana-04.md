# 🎯 Semana 04 - Portas TCP vs UDP, Forewalls e NAT

Fase: 1 - Fundamentos

---

## O que estudei essa semana

### TCP vc UDP

| | TCP | UDP |
|---|---|---|
| Conexão | Orientado a conexão (handshake) | Sem conexão |
| Condfiabilidade | Confirma entrega, reenvia perdas, ordena pacotes | Sem garantia de entrega ou ordem |
| Custo | Mais pacotes e mais overhead | Leve e rápido |
| Usos | HTTP/S, SSH, tranferência de arquivos | DNS, DHCP, streaming, VoIP |

### Three-way handshake (TCP)

Cliente -> Servidor: SYN (posso conectar?)
Servidor -> Cliente: SYN-ACK (pode, confirmado)
Cliente -> Servidor: ACK (feechado, vamos conversar)
Só depois disso os dados trafegam. No fim da conversa aparecem pacotes de encerramento (FIN ou RST).
..
### Firewalls

- Stateless: avalia cada pacote isoladamente, só pelas regras fixas (IP,porta).
- Stateful: manteḿ tabela de conexões ativas. Se a conversa foi iniciada de dentro, a resposta que volta é permitida sem regra explícita.
- Regras de entrada (inbound) e saída (outbound) são avaliadas separadamente.

### NAT (Network Address Translation)

- Traduz endereços IP privados em um IP de saída, permitindo que várias máquinas compartilhem uma única conexão.
- NAT estático: 1 IP privado <-> 1 IP público
- NAT dinâmico/PAT: vários IPs privados compartilhando 1 IP, diferenciados por porta (o mais comum).
- No VirtualBox, o modo NAT roda um DHCP interno próprio e entrega à VM um IP de uma rede privada (10.0.2.x no adaptador 1, 10.0.3.x no adaptador 2, e assim por diante). O VirtualBox traduz o tráfego para o IP real do host na saída.

--- 

## 🧪 Exercícios práticos no lab

### Exercício  - Three-way handshake no Wireshark

No Kali, com filtro tcp e a captura rodando:
curl -k https://192.168.56.20
Resultado: vi o cliente enviar SYN, o servidor responder SYN-ACK e o cliente confirmar com ACK, iniciando a conexão.

### Exercício 2 - TCP vs UDP no Wireshark

Com filto UDP.port == 53:
nslookup lab.local 192.168.56.10
Resultado: no UDP não aparerem flags de handshake. A troca é direta (pergunta e resposta, 2 pacotes), sem verificação de conexão. COntando handshake, dados e encerramento, o TCP usa cerca de 10 pacotes ou mais para uma conversa equivalente.

### Exercício 3 - Firewall stateful em ação

Reativei o firewall do WIN11-CLIENT e testei ping a partir do Kali:
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled True
Resultado: o firewall bloqueou o ping. O Comportamento foi um timeout silencioso, e não um erro de "unreachable": o pacote chegou à rede certa e o destino descartou o ICMP de entrada.

Depois desativei de novo para não atrapalhar o resto do lab.

### Exercício 4 - De nde veio o IP 10.0.3.15

Pergunta: por que o DC01 recebeu um IP tipo 10.0.3.15 quando usei o adaptador NAT temporário?

Resposta:
- Ao ligar o NAT, o VirtualBox entrega à VM um IP de uma rede privada própria, que só existe entre a VM e VirtualBox. O padrão é 10.0.3.x no segundo adaptador.
- Um Domain Controller registra dinamicamente os IPs de todas as suas placas na zona DNS. O 10.0.3.15 foi registrado assim.
_ Ao desligar o adaptador NAT, o registro continuou na zona lab.local, apontando para uma rede inacessível pelo lab. Foi isso que causou o timeout de DNS da Semana 03.

### Exercício 5 - closed vc filtered no nmap

nmap -p 9200 192.168.56.20
nmap -p 3389 192.168.56.11
Resultado: com firewall ativo, o nmap mostra a porta como filtered.

| Estado | Significado |
|---|---|
| closed | A máquina respondeu com RST: está viva e não há seviço naquela porta |
| filtered | Nenhuma resposta voltou: algo no caminho (geralmente firewall) descartou o pacote |

---

## 💡 Lições aprendidas

1. TCP e UDP existem por motivos opostos: TCP prioriza integridade, UDP prioriza velocidade. DNS e DHCP usam UDP porque a troca é pequena e um handshake custaria mais que a conversa.
2. Um scan com muitos filtered revela a presença de firewall. Do lado de defesa, o mesmo scan gera uma rajada de conexões bloqueada nos logs, um padrão clássico de detecção de reconhecimento.
3. COnfigurações temporárias deixam rastros. O NAT que liguei por alguns minutos deixou um resgistro órfão no DNS que só apareceu semanas depois.
4. Timeout silencioso e erro explícito indicam problemas diferentes: silêncio sugere descarte (firewall ou host ausente), erro de rota sugere configurações de rede.

---

## ✅ Checklist da semana

- [x] Entendi a diferença estrutural entre TCP e UDP
- [x] Vi o three-way handshake acontecendo no Wireshark
- [x] Entendi a diferença entre firewall stateful e stateless
- [x] Entendi como o NAT funciona e cnectei com o bug real da Semana 
- [x] Fiz os  exercícios práticos no lab
- [x] Fiz o quiz de fixação (resultado: 5/5)
- [x] Documentei aqui