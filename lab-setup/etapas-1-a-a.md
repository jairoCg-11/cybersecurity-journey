# Etapas 1 a 4 — Setup completo do Lab

## Etapa 1 — Rede isolada (VirtualBox)

**Objetivo:** criar uma rede que só as VMs do lab enxergam, sem contato com a rede real.

1. VirtualBox → Preferences → Network → Host-only Networks → Add
2. Nome: `vboxnet0`, DHCP habilitado

⚠️ Nenhuma VM usa "Bridged Adapter" — sempre "Host-only Adapter".

---

## Etapa 2 — Domain Controller (DC01)

### Criação da VM
- Windows Server 2022, 4096MB RAM, 60GB disco, rede Host-only

### IP fixo
```powershell
# Configurado via interface gráfica:
# IP: 192.168.56.10 | Máscara: 255.255.255.0 | Gateway: vazio | DNS: 127.0.0.1
```
*Por quê DNS 127.0.0.1: o próprio DC vai virar o servidor DNS do domínio.*

### Instalar a role Active Directory Domain Services
```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
```
*O que faz: instala os componentes necessários para o servidor atuar como controlador de domínio.*

### Promover a Domain Controller
```powershell
Import-Module ADDSDeployment

Install-ADDSForest `
  -DomainName "lab.local" `
  -DomainNetbiosName "LAB" `
  -InstallDns:$true `
  -SafeModeAdministratorPassword (ConvertTo-SecureString "SENHA_AQUI" -AsPlainText -Force) `
  -Force:$true
```
*O que faz: cria uma nova floresta AD com o domínio `lab.local`, instala o serviço DNS junto, e define a senha de recuperação (DSRM).*

### Erros enfrentados e soluções
- **Erro `Test.VerifyDcport`:** causado por firewall ativo bloqueando a checagem de portas. Resolvido desativando o firewall (lab isolado, sem risco):
```powershell
  Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False
```

### Verificação final
```powershell
Get-ADDomain
```
*Confirma que o domínio `lab.local` está ativo.*

---

## Etapa 3 — Cliente Windows (WIN11-CLIENT)

### Criação da VM
- Windows 11 Enterprise (Evaluation), 4096MB RAM, 50GB disco, rede Host-only
- **Observação:** foi necessário fazer bypass do TPM/Secure Boot durante a instalação, pois o VirtualBox não validava esses requisitos corretamente.

### IP fixo

IP: 192.168.56.11 | DNS: 192.168.56.10 (aponta para o DC, não para 127.0.0.1)


### Entrar no domínio
- Configurações avançadas do sistema → Change → Member of: Domain → `lab.local`
- Autenticação com `LAB\Administrator`

### Verificação
```powershell
whoami
# retorna: lab\administrator

Get-ComputerInfo | Select CsDomain
# retorna: lab.local
```

### Instalar Sysmon
```powershell
.\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
```
*O que faz: instala o Sysmon (monitor de atividade do sistema) usando a configuração da comunidade SwiftOnSecurity, que já vem otimizada para detectar atividades suspeitas.*

```powershell
Get-Service -Name Sysmon64
# confirma status "Running"
```

---

## Etapa 4 — SIEM (SIEM01 + Wazuh)

### Criação da VM
- Ubuntu Server 24.04 LTS, inicialmente 4096MB RAM / 40GB disco (depois aumentado — ver "Problemas enfrentados")
- **Dois adaptadores de rede:** NAT (internet, para baixar pacotes) + Host-only (rede do lab)

### IP fixo (via Netplan)
```bash
sudo tee /etc/netplan/00-installer-config.yaml > /dev/null << 'EOF'
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      dhcp6: false
      addresses:
        - 192.168.56.20/24
      match:
        macaddress: 08:00:27:e7:c5:4e
      set-name: enp0s3
    enp0s8:
      accept-ra: true
      dhcp4: true
      dhcp6: true
EOF

sudo netplan apply
```
*O que faz: fixa o IP `192.168.56.20` na interface host-only (`enp0s3`), mantendo a outra interface (`enp0s8`) com DHCP para acesso à internet via NAT.*

### Instalação do Wazuh
```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash wazuh-install.sh -a
```
*O que faz: baixa e executa o instalador "all-in-one" do Wazuh, que instala e configura o indexer, manager e dashboard juntos no mesmo servidor.*

### Problemas enfrentados e soluções

**1. RAM insuficiente:** a instalação exigia no mínimo 4GB livres. Aumentado de 4GB para ~5GB, depois oito.

**2. Disco cheio ("disk full error"):** o disco de 40GB não comportou os 3 componentes do Wazuh. Solução: redimensionar o disco virtual sem reinstalar o Ubuntu.

```bash
# No Mac (host), localizar o disco:
VBoxManage list hdds

# Redimensionar usando o UUID (evita problemas com espaços/acentos no caminho):
VBoxManage modifymedium disk <UUID_DO_DISCO> --resize 81920
```
*O que faz: expande o arquivo .vdi de 40GB para 80GB no nível do VirtualBox.*

```bash
# Dentro do Ubuntu, expandir a partição física:
sudo growpart /dev/sda 3

# Expandir o volume físico do LVM:
sudo pvresize /dev/sda3

# Verificar espaço livre no volume group:
sudo vgs

# Estender o volume lógico para usar todo o espaço livre:
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv

# Redimensionar o filesystem para ocupar o novo espaço:
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```
*Por quê essa sequência: o Ubuntu usa LVM (gerenciador de volumes lógicos) por padrão, então expandir só a partição física não é suficiente — é preciso propagar o crescimento por três camadas: partição → volume físico → volume lógico → filesystem.*

### Recuperar credenciais geradas na instalação
```bash
sudo tar -xf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt -O
```
*O que faz: extrai o arquivo de senhas geradas automaticamente durante a instalação (usuário admin do dashboard, senhas internas de outros componentes).*

### Acesso ao dashboard


https://192.168.56.20
Usuário: admin
Senha: (gerada na instalação)


---

## Conectar o agente Wazuh no WIN11-CLIENT

### Baixar e instalar o agente (rodado no WIN11-CLIENT)
```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.7-1.msi -OutFile $env:tmp\wazuh-agent.msi

msiexec.exe /i $env:tmp\wazuh-agent.msi /q WAZUH_MANAGER="192.168.56.20" WAZUH_AGENT_NAME="WIN11-CLIENT"
```
*O que faz: baixa o instalador do agente Wazuh e instala silenciosamente (`/q`), já configurado para se comunicar com o SIEM no IP `192.168.56.20`.*

### Iniciar o serviço
```powershell
NET START WazuhSvc
```

### Verificação final
No dashboard do Wazuh: Agents management → Summary → `WIN11-CLIENT` aparece com status **Active** (verde), IP `192.168.56.11`, SO detectado corretamente.

---

## ✅ Resultado final

Lab funcional com:
- Domain Controller (`lab.local`)
- Cliente Windows 11 monitorado por Sysmon
- SIEM (Wazuh) recebendo e indexando eventos em tempo real

## Lições aprendidas
- Erros de infraestrutura (rede, disco, TPM) são parte normal do processo e ensinam tanto quanto o resultado final
- LVM exige expansão em camadas (partição → PV → LV → filesystem)
- Sempre verificar a versão correta nas URLs de download (URLs genéricas tipo `/4.x/` podem retornar erro 403 dependendo do momento)