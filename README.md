# Pipeline Tributário

**Automação e Escalabilidade de Pipelines de Big Data e Análise de Dados**
*Diagnóstico Automatizado de Enquadramento Tributário para Consultoria Contábil*

Pipeline que compara a carga tributária estimada de empresas nos regimes Simples Nacional, Lucro Presumido e Lucro Real. Este repositório contém a infraestrutura como código que provisiona e configura a máquina virtual onde o simulador de dados é executado.

| Etapa | Ferramenta | Função |
|---|---|---|
| Provisionamento | OpenTofu + libvirt/KVM | Cria a VM Ubuntu 24.04 e gera o inventário do Ansible |
| Configuração inicial | cloud-init | Usuário, acesso SSH por chave e hostname |
| Preparação do ambiente | Ansible | Python, ambiente virtual, dependências e diretórios |

## Estrutura

```
infraestrutura/
├── main.tf, variables.tf     # provisionamento da VM
├── cloud_init.cfg            # configuração do primeiro boot
└── ansible/
    ├── inventory.ini         # gerado automaticamente pelo OpenTofu
    └── playbook.yml          # preparação do ambiente
simulador/                    # código do simulador
dados/                        # exemplo dos dados gerados
```

## Pré-requisitos

Máquina hospedeira Linux com virtualização ativada na BIOS (testado em Ubuntu 26.04):

```bash
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients virtinst genisoimage ansible
sudo adduser "$USER" libvirt && sudo adduser "$USER" kvm   # depois, encerre a sessão e entre de novo
sudo virsh net-autostart default && sudo virsh net-start default

curl --proto '=https' --tlsv1.2 -fsSL https://get.opentofu.org/install-opentofu.sh -o install-opentofu.sh
chmod +x install-opentofu.sh && ./install-opentofu.sh --install-method standalone --skip-verify
rm -f install-opentofu.sh
```

Nenhuma credencial é versionada. Gere uma chave SSH e baixe a imagem do Ubuntu:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/vm_pipeline -N ""
mkdir -p ~/imagens-vm
wget -P ~/imagens-vm https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
```

Caminhos diferentes podem ser informados com `-var`, conforme `infraestrutura/variables.tf`.

## Execução

```bash
cd infraestrutura

# 1. Provisionar a VM
tofu init
tofu apply

# 2. Verificar o cloud-init (aguarde cerca de um minuto)
ssh -i ~/.ssh/vm_pipeline pipeline@$(tofu output -raw ip_vm)
cloud-init status                       # status: done
cat /var/log/cloud-init-pipeline.log    # data de aplicação
exit

# 3. Preparar o ambiente
ansible -i ansible/inventory.ini simulador -m ping
ansible-playbook -i ansible/inventory.ini ansible/playbook.yml
```

O playbook é idempotente: uma segunda execução termina com `changed=0`. Ele cria `/opt/simulador` para o código e `/var/lib/simulador/dados` para os dados, mantendo os dois separados.

## Reconstrução

```bash
tofu destroy
tofu apply
ssh-keygen -R "$(tofu output -raw ip_vm)"    # remove a chave de host da VM anterior
ansible-playbook -i ansible/inventory.ini ansible/playbook.yml
```

## Solução de problemas

**`Permission denied` em arquivo `.qcow2` no `tofu apply`:** o AppArmor bloqueia o disco da VM. Aceitável apenas em laboratório local:

```bash
echo 'security_driver = "none"' | sudo tee -a /etc/libvirt/qemu.conf
sudo systemctl restart libvirtd
```

**`qemu-kvm` não encontrado:** em versões recentes do Ubuntu, use `qemu-system-x86`.

**Ansible `UNREACHABLE` após o `apply`:** o cloud-init ainda está em execução. Aguarde e tente novamente.
