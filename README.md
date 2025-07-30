# Documentação Provisionamento Automático Proxmox/Terraform

## Instalação Proxmox


Primeiro é preciso instalar proxmox através da iso, é só fazer o download no site oficial e instalar como qualquer outro sistema operacional.

É preciso habilitar a virtualização na bios da máquina se não estiver habilitada já.

Após a instalação do sistema operacional é preciso seguir os passos abaixo para criar uma máquina virtual padrão para os clones.

instalar pacote:
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

Se o proxmox for instalada no VirtualBox para testes, as seguintes configurações são necessárias:

Executar o código abaixo na pasta de instalação do virtualbox, isso habilita a virtualização aninhada:
`./VBoxManage modifyvm Proxmox --nested-hw-virt on`

para ligar a máquina, desativar virtualização no proxmox (em options) e setar cpu:
`qm set 100 --cpu=kvm64`
