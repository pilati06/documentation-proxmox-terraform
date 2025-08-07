# Lab Virtual - Sistema de Provisionamento Automático de VMs

Sistema automatizado para provisionamento de máquinas virtuais através de tickets de solicitação com aprovação e expiração automática.

## 📋 Visão Geral

O Lab Virtual é uma solução completa que permite aos usuários solicitarem recursos computacionais (CPU, RAM, espaço em disco e tempo de disponibilidade) através de um frontend web. Após aprovação, o sistema automaticamente provisiona máquinas virtuais usando Terraform e Proxmox.

## 🏗️ Arquitetura

```
Frontend → API (Oracle APEX) → Banco de Dados → VM Core (Scheduler) → Terraform → Proxmox VE
```

### Componentes Principais

- **Frontend**: Interface web para solicitação de recursos
- **API Backend**: Oracle APEX com autenticação Basic Auth
- **VM Core**: Script scheduler que monitora tickets aprovados
- **Terraform**: Automação de infraestrutura como código
- **Proxmox VE**: Plataforma de virtualização

## 🚀 Workflow

1. **Solicitação**: Usuário cria ticket via formulário especificando recursos necessários
2. **Aprovação**: Ticket é submetido para aprovação manual
3. **Provisionamento**: Sistema cria VM automaticamente após aprovação
4. **Notificação**: Usuário recebe credenciais de acesso via email
5. **Expiração**: VM é removida automaticamente na data de expiração

## 🛠️ Pré-requisitos

### Sistema Base
- Habilitar virtualização na BIOS da máquina
- Download da ISO oficial do Proxmox VE
- Instalação do Proxmox como sistema operacional base

### VirtualBox (para desenvolvimento)
```bash
# Habilitar virtualização aninhada
./VBoxManage modifyvm Proxmox --nested-hw-virt on
```
> Configurar placa de rede em modo bridge para acesso via navegador

### Máquina Core (VM Scheduler)

É uma máquina virtual criada no proxmox. Essa VM Core serve de base para as automações. Os scripts de criação/destruição de vms e envio de emails estão rodando nessa vm.

- Sistema Linux com acesso à API
- Terraform instalado
- Cron jobs configurados
- Acesso SSH ao Proxmox

### Proxmox VE

#### Instalação e Configuração
```bash
# Atualizar sistema e instalar dependências
apt-get update
apt install libguestfs-tools -y

# Download da imagem Ubuntu Cloud
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img

# Instalar qemu-guest-agent na imagem
virt-customize --add noble-server-cloudimg-amd64.img --install qemu-guest-agent
```

#### Criação do Template
```bash
# Criar VM template
qm create 9001 --name ubuntu-2404-cloud-init --numa 0 --ostype l26 --cpu cputype=host --cores 2 --sockets 2 --memory 2048 --net0 virtio,bridge=vmbr0

# Importar disco para storage
qm importdisk 9001 /tmp/noble-server-cloudimg-amd64.img local-lvm

# Configurar storage
qm set 9001 --scsihw virtio-scsi-pci --scsi0 local-lvm:9001:vm-9001-disk-0

# Configurar Cloud-init
qm set 9001 --ide2 local-lvm:cloudinit

# Definir disco de boot
qm set 9001 --boot c --bootdisk scsi0

# Configurar console serial
qm set 9001 --serial0 socket --vga serial0

# Habilitar guest agent
qm set 9001 --agent enabled=1

# Expandir disco (opcional)
qm disk resize 9001 scsi0 +4G
```

#### Verificação do Sistema
```bash
# Verificar informações da CPU (cores disponíveis)
cat /proc/cpuinfo

# Para VMs em VirtualBox, configurar CPU
qm set 100 --cpu=kvm64
```

### Terraform

#### Instalação
Seguir as instruções oficiais: https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli

Para utilizar o terraform é aconselhavel gerar um token de api no proxmox, este tutorial define os passo para gerar: [https://www.youtube.com/watch?v=1kFBk0ePtxo](https://www.youtube.com/watch?v=1kFBk0ePtxo)

As seguintes permissões devem ser habilitadas para o token:

<img width="299" height="615" alt="Captura de tela 2025-07-23 171517" src="https://github.com/user-attachments/assets/0954dfaf-6f8f-4e9f-a0d4-cdaf21522001" />

#### Configuração dos Templates

Consultar o padrão no repositório - **Terraform Templates**: Configurações de infraestrutura ([pilati06/terraform](https://github.com/pilati06/terraform))

Deve-se clonar esse repositório na máquina core.

## 🔄 Scripts de Automação

### VM Scheduler Script

Consultar os scripts de automação no repositório:

- **VM Scheduler**: Scripts de automação e scheduling ([pilati06/vm-scheduler](https://github.com/pilati06/vm-scheduler))

Também deve ser clonado na máquina core.

### Cron Jobs

```bash
# Editar crontab
crontab -e

# Verificar tickets aprovados a cada 5 minutos
*/5 * * * * /root/vm-scheduler/generate-vm-info.sh 2>&1 | ts "%Y-%m-%d %H:%M:%S" >> /root/logs/$(date +%Y-%m-%d).log

# Enviar emails com credenciais a cada 5 minutos
*/5 * * * * /root/vm-scheduler/sendmails-vm.sh 2>&1 | ts "%H:%M:%S" >> /root/logs/$(date +%Y-%m-%d)-mail.log

# Limpeza de logs antigos (diário às 2h)
0 2 * * * find /root/logs -type f -mtime +90 -name "*.log" -exec rm -f {} \;
```

## 🐳 Integração com Docker

### Cloud-Init para Docker
```yaml
#cloud-config
package_update: true
package_upgrade: true
packages:
  - docker.io
  - docker-compose
users:
  - name: ubuntu
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
ssh_pwauth: true
chpasswd:
  list: |
    ubuntu:${password}
  expire: true
runcmd:
  - sed -i 's/^PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
  - systemctl restart ssh
  - systemctl enable docker
  - systemctl start docker
  - usermod -aG docker ubuntu
```

### Configuração de Usuários
```yaml
#cloud-config
users:
  - name: ubuntu
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
ssh_pwauth: true
chpasswd:
  list: |
    ubuntu:${password}
  expire: true
runcmd:
  - sed -i 's/^PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
  - systemctl restart ssh
```

## 🔐 Configuração de SSH

```bash
# Gerar chaves SSH para Terraform
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa_proxmox -C "terraform@proxmox"

# Copiar chave pública para Proxmox
ssh-copy-id -i ~/.ssh/id_rsa_proxmox.pub root@PROXMOX_IP
```

## 📊 Estrutura da API Response

### Endpoint de Tickets Aprovados

Exemplo de resposta:

```json
{
  "items": [
    {
      "id": 25,
      "ano": 2025,
      "memoria": 1,
      "disco": 4,
      "nucleos": 1,
      "tempo_uso": 2,
      "descricao": "teste",
      "res_incl": "USER",
      "dat_incl": "2025-07-30T00:00:00Z",
      "dat_fim": "2025-08-01T18:31:25Z",
      "task_id": "69704933938847278031",
      "orientador": null,
      "memoria_calc": 1024,
      "docker": 0
    }
  ]
}
```

### Campos da Response

| Campo | Descrição | Tipo |
|-------|-----------|------|
| `id` | ID único do ticket | Integer |
| `ano` | Ano de criação | Integer |
| `memoria` | Memória solicitada (GB) | Integer |
| `disco` | Espaço em disco (GB) | Integer |
| `nucleos` | Número de CPUs | Integer |
| `tempo_uso` | Tempo de uso (dias) | Integer |
| `memoria_calc` | Memória calculada (MB) | Integer |
| `docker` | Flag para suporte Docker | Integer (0/1) |
| `dat_incl` | Data de criação | ISO DateTime |
| `dat_fim` | Data de expiração | ISO DateTime |

## 📊 API Endpoints

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| GET | `/labvirtual/tasks/approved` | Buscar tickets aprovados |
| POST | `/labvirtual/sendmail` | Atualizar status da VM |

Enviar post com o seguinte corpo:

```json
{ 
  "vmname": "NOME_VM", 
  "password": "PRIMEIRA_SENHA_SEGURA", 
  "ipvm": "IP_MAQUINA", 
  "user": "USER" 
}
```

## 📁 Estrutura do Projeto

```
lab-virtual/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   └── terraform.tfvars
│   └── cloudinit-config-files
│       ├── cloudinit-docker.yaml.tmpl
│       └── cloudinit.yaml.tmpl
├── vm-scheduler/
│   ├── generate-vm-info.sh
│   └── sendmails-vm.sh
├── logs/
│   └── [Arquivos de log]
```

## 🔗 Repositórios Relacionados

Este projeto faz parte de um ecossistema maior com os seguintes repositórios:

- **VM Scheduler**: Scripts de automação e scheduling ([pilati06/vm-scheduler](https://github.com/pilati06/vm-scheduler))
- **Terraform Templates**: Configurações de infraestrutura ([pilati06/terraform](https://github.com/pilati06/terraform))
- **Documentação Oficial**: [pilati06.github.io/documentation-proxmox-terraform](https://pilati06.github.io/documentation-proxmox-terraform/)

### Providers Terraform Recomendados

Para diferentes necessidades, considere os seguintes providers:

- **Telmate/proxmox** (usado neste projeto): Provider estável e amplamente utilizado
- **bpg/proxmox**: Provider mais moderno com recursos adicionais
- **danitso/proxmox**: Alternativa com funcionalidades específicas

## 📈 Recursos Avançados

### Monitoramento e Logs
- Logs estruturados com timestamps
- Limpeza automática de arquivos antigos
- Monitoramento do status das VMs
- Alertas por email sobre falhas

### Escalabilidade
- Suporte a múltiplas VMs simultâneas
- Gerenciamento de recursos por quotas
- Balanceamento de carga entre nós Proxmox

### Integração
- API RESTful completa
- Webhooks para notificações
- Integração com sistemas de ticketing

### Dependências Necessárias
```bash
# HTTPie para requisições HTTP
sudo apt install httpie

# moreutils para timestamp nos logs
sudo apt install moreutils

# jq para parsing JSON
sudo apt install jq
```

## ⚠️ Configurações de Segurança

- Alterar credenciais padrão antes da produção
- Configurar firewall adequadamente
- Usar HTTPS em produção
- Implementar rotação de tokens da API
- Configurar backup regular dos dados

## 📝 Logs

Os logs são armazenados em `/root/logs/` com limpeza automática de arquivos com mais de 90 dias.
