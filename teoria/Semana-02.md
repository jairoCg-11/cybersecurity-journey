# Semana 02 - Endereçamento IP e Subnetting

Fase: 1 - Fundamentos

--- 

- Estrutura de um IPv4 (4 octetos) e faixas privadas (RFC 1918) - meu lab usa 192.168.56.0/24
- Notação CIDR (/24, /26, etc.) vs máscara decimal (255.255.255.0)
- Cálculo de: endereço de rede, broadcast, primeiro/último host utilizável
- Divisão de uma rede maior em sub-redes menores (Divider da maior para menor)
- Conceito de gateway padrão e por que nem toda VM do meu lab precisa de um

### Método de cálculo (fixado)

Para qualquer bloco de sub-rede, respondo nessa ordem:
1. Quantos bits foram emprestados da parte de host?
2. Qual o salto entre blocos? (256 - último octeto da máscara)
3. Em qual bloco meu IP se encaixa? (IP dividido pelo salto, arrendodado para baixo, multiplicar de volta)
4. Rede = início do bloco|Broadcast = próximo bloco - 1|Primeiro host=rede+1|Último host=broadcast-1

Tabela de referência rápida:

| CIDR | Máscara (último octeto) | Salto | Host utilizáveis |
|---|---|---|---|
| /24 | .0 | 256 | 254 |
| /25 | .128 | 128 | 126 |
| /26 | .192 | 64 | 62 |
| /27 | .224 | 32 | 30 |
| /28 | .240 | 16 | 14 |
| /29 | .248 | 8 | 6 | 
| /30 | .252 | 4 | 2 |

### Gateway
O gateway padrão é p dispositivo usado quando uma máquina precisa se comunicar com um IP fora da sua própria sub-rede. No meu lab, DC01, WIN11-CLIENT e Kali não têm gateway configurado (só conversam entre si, mesma sub-rede). O SIEM01 tem uma segunda placa de rede em NAT justamente para poder acessar a internet quando necessário (atualizações).

---

## Exercícios práticos no lab

### Execício 1 - Mapear minha própria rede (192.168.56.0/24)
- Endereço de rede: 192.168.56.0
- Broadcast: 192.168.56.255
- Primeiro host: 192.168.56.1
- Último host: 192.168.56.254
- Total de hosts utilizáveis: 254

### Exercício 2 - Conferir máscara em cada VM
Confirmado via ip a (Linux) e ipconfig /all (Windows) que todas as VMs estão na mesma sub-rede /24.

### Exercício 3 - Ip fora vs máscara da sub-rede
Testando ping para:
- 192.168.57.5 (fora sub-rede) -> não há rota configurada, sistema não consegue nem tentar entragar o pacote ("network unreachable")
- 198.162.56.99 (dentro da sub-rede, sem host) -> é enviado a ARP request, mas ninguém responde -> timeout silenciado

Diferença-chave: erro de rota (fora da sub-rede) é diferente de timeout por ausência de host (dentro da sub-rede) - útil para diagnosticar problemas de rede dia a dia.

### Exercício 4 - Cálculos manuais de subnetting

Dividir 192.168.56.0/24 em 4 sub-redes iguais:
- Máscara: /26 (255.255.255.192)-
- Blocos: 192.168.56.0, .64, .128, .192

Hosts utilizáveis numa /28: 14 (16 endereços totais - 2 reservados)

Bloco livres para a 5ª VM (sem conflitar com .10, .11, .20)
192.168.56.32/28 (faixa .32-.47) - o bloco .16/28 (faixa .16-.31) contém o SIEM01 (.20) e não pode ser usada.

---

## Drill extra de fixação (identificação e blocos)

Pratiquei um rodada adicional de exercícios focando especificamente em identificar o bloco correto a partir de um IP + máscara, usando o método IP divido pelo salto -> arrendodar para baixo -> multiplicar de volta.

--- 

## Checklist da semana

- [x] Entendi a diferença entre endereço de rede, broadcast e host
- [x] Sei converter notação CIDR para máscara decimal e vire e versa
- [x] Consigo calcular manualmente o primeiro/último host de uma sub-rede
- [x] Entendi o conceito de gateway e por que nem toda VM do lab precisa de um
- [x] Fiz os 4 exerícios práticos no lab
- [x] Pratiquei drill de identificação de blocos
- [x] Documentei aqui