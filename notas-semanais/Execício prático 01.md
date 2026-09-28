Execício prático 01
Obs: Ligue as Vm's e ao pesquisar os enreços MAC's não pareceram pois ainda os computadores não tinha se 'conversados', pois a tabela arp so armazena quando se conversam.

1. Camada 2 -ss Enlace - endereços MAC
rodei arp -a segue os endereços:
DC01=08:00:27:d7:04:b1
WIN11-CLIENT=08:00:27:40:7f:c6 
SIEM01=08:00:27:e7:c5:4e


2. Camada 3 (Rede) - Endereços e rota
Para verificar o ip (ip a) e rota (ip route).

Ip do Kali = 192.168.56.7
Gateway = nenhum (Kali só conversa na própria sub-rede)
Rota = 192.168.56.0/24 (rede local, diretamente conectada)

3. Camada 4  (Transporte) TCP vs UDP
Para ver portas aberta no proprio kali (ss -tulpn). Aparece se tiver algum serviço rodando.

4. Camada 7 (Aplicação) Testa o protocolo real. curl -k https://192.168.56.20
