# Documentação Provisionamento Automático Proxmox/Terraform

## Instalação Proxmox

Primeiro é preciso instalar proxmox através da iso, é só fazer o download no site oficial e instalar como qualquer outro sistema operacional.

É preciso habilitar a virtualização na bios da máquina se não estiver habilitada já.

Após a instalação do sistema operacional é preciso seguir os passos abaixo para criar uma máquina virtual padrão para os clones.

instalar pacote libguestfs-tools:

`apt-get update`

`apt install libguestfs-tools -y`

Baixar img do OS cloud (ex.: ubuntu server cloud img) com wget.

Instalar qemu-guest-agent na img com:

`virt-customize –add noble-server-cloudimg-amd64.img –install qemu-guest-agent`

É possível verificar informações do cpu (como cores) com o comando:
`cat /proc/cpuinfo`

Criar máquina virtual template:
`qm create 9001 --name ubuntu-2404-cloud-init --numa 0 --ostype l26 --cpu cputype=host --cores 2 --sockets 2 --memory 2048 --net0 virtio,bridge=vmbr0`

Importar img para o storage:
`qm importdisk 9001 /tmp/noble-server-cloudimg-amd64.img local-lvm`

Colocar a VM para utilizar o storage:
`qm set 9001 –scsihw virtio-scsi-pci –scsi0 local-lvm:9001:vm-9001-disk-0`

Configurar Cloud-init:
`qm set 9001 –ide2 local-lvm:cloudinit`

Definir scsi0 como disco de boot:
`qm set 9001 –boot c –bootdisk scsi0`

Configurar socket:
`qm set 9001 –serial0 socket –vga serial0`

Habilitar guest agent:
`qm set 9001 –agent enabled=1`

Aumentar tamanho do disco (Opcional):
`qm disk resize 9001 scsi0 +4G`

converte para template
`qm template 9001`

### Se o proxmox for instalado no VirtualBox para testes, as seguintes configurações são necessárias:

Executar o código abaixo na pasta de instalação do virtualbox, isso habilita a virtualização aninhada:
`./VBoxManage modifyvm Proxmox --nested-hw-virt on`

para ligar a máquina, desativar virtualização no proxmox (em options) e setar cpu:
`qm set 100 --cpu=kvm64`

## Máquina virtual Core

Para realizar a automação da criação e destruição das máquinas virtuais foi criado um script que verifica um endpoint onde a resposta é como no código abaixo:

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
  ],
}
```

Após a verificação dessa resposta, o script cria um arquivo que vai servir de consulta para o terraform sincronizar a criação de máquinas virtuais no proxmox.

O repositório para esse script é o seguinte: [https://github.com/pilati06/vm-scheduler](https://github.com/pilati06/vm-scheduler)

No final do script o terraforma é executado para criar/destruir automaticamente as vms.

## Terraform

O terraform precisa ser instalado na máquina Core: [https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)

O repositório do terraform é o seguinte: [https://github.com/pilati06/terraform](https://github.com/pilati06/terraform)

Para utilizar o terraform é aconselhavel gerar um token de api no proxmox, este tutorial define os passo para gerar: [https://www.youtube.com/watch?v=1kFBk0ePtxo](https://www.youtube.com/watch?v=1kFBk0ePtxo)

As seguintes permissões devem ser habilitadas para o token:

<img width="299" height="615" alt="Captura de tela 2025-07-23 171517" src="https://github.com/user-attachments/assets/0954dfaf-6f8f-4e9f-a0d4-cdaf21522001" />

## Cron Job

Na máquina Core é preciso criar um cron job para executar o script bash.
Execute: `crontab -e`

e em seguida adicione a linha:

`*/5 * * * * /root/vm-scheduler/generate-vm-info.sh 2>&1 | ts "%Y-%m-%d %H:%M:%S" >> /root/logs/$(date +%Y-%m-%d).log`

Essa linha cria um scheduler que executa o script a cada 5 minutos, imprimindo o log de saída num arquivo com o nome da data atual, e com timestamp em cada linha. O comando `ts` é do pacote moreultils para imprimir tag de horário no log.
 
