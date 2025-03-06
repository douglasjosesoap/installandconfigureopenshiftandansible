# Passo a Passo: Instalação e Configuração do OpenShift e Ansible na Nuvem (AWS, Azure, GCP)
Este guia detalha o processo de implantação do OpenShift e do Ansible nas três principais nuvens (AWS, Azure e GCP). Vamos configurar OpenShift em cada provedor e conectá-lo ao Ansible para automação.

# 1. Escolhendo a Infraestrutura
Vamos criar máquinas virtuais (VMs) em AWS, Azure e Google Cloud (GCP) para rodar o OpenShift e instalar o Ansible em outra máquina separada.

# Especificações da Máquina Virtual para OpenShift
vCPUs: 8
RAM: 32GB
Disco: 150GB SSD
SO: Red Hat Enterprise Linux 9 (RHEL 9)
Rede: IP público + firewall liberado para portas OpenShift
📌 Especificações da Máquina Virtual para Ansible
vCPUs: 4
RAM: 16GB
Disco: 50GB SSD
SO: Red Hat Enterprise Linux 9 (RHEL 9)
Rede: Acesso SSH e conectividade com o OpenShift

# 2. Criando Máquinas Virtuais para OpenShift em AWS, Azure e GCP
# 2.1 Criando a VM na AWS

Acesse a AWS Console: https://aws.amazon.com/
Vá para EC2 > Instances > Launch Instance
Escolha Red Hat Enterprise Linux 9
Escolha o tipo de instância: m5.2xlarge (mínimo 8 vCPUs e 32GB RAM)
Configure o armazenamento: 150GB SSD
Configure a segurança:
Porta 22 (SSH) → Acesso público ou IP específico
Porta 6443 (API OpenShift) → IP da VM Ansible
Porta 80, 443 (Acesso Web OpenShift)
Crie ou selecione uma Key Pair para SSH
Clique em Launch Instance
Após a criação, copie o IP público da VM
Conecte-se à VM via SSH:

ssh -i chave.pem ec2-user@<IP_PÚBLICO>

# 2.2 Criando a VM na Azure
Acesse Azure Portal: https://portal.azure.com/
Vá para Máquinas Virtuais e clique em Criar
Escolha Red Hat Enterprise Linux 9
Escolha o tamanho: Standard D8s v5 (8 vCPUs, 32GB RAM)
Configure 150GB SSD
Configure a rede e segurança:
Porta 22 (SSH) → Acesso público ou IP específico
Porta 6443 (API OpenShift) → IP da VM Ansible
Porta 80, 443 (Acesso Web OpenShift)
Gere um par de chaves SSH
Clique em Criar
Conecte-se via SSH:
ssh -i chave.pem azureuser@<IP_PÚBLICO>

# 2.3 Criando a VM na GCP
Acesse Google Cloud Console: https://console.cloud.google.com/
Vá para Compute Engine > Instâncias de VM
Clique em Criar Instância
Escolha Red Hat Enterprise Linux 9
Escolha o tipo n2-standard-8 (8 vCPUs, 32GB RAM)
Configure 150GB SSD
Configure a rede e firewall:
Permitir tráfego TCP 22, 6443, 80, 443
Clique em Criar
Conecte-se via SSH:
gcloud compute ssh <NOME_DA_INSTANCIA> --zone=<ZONA>

# 3. Instalando e Configurando OpenShift
Agora que as VMs estão criadas, vamos instalar o OpenShift em cada uma.

# 3.1 Atualizar e Preparar o Ambiente
Em cada VM, execute:

sudo dnf update -y
sudo dnf install -y git curl wget vim tar net-tools bind-utils yum-utils jq

# 3.2 Instalar o OpenShift CLI (oc)
Baixe e instale o OpenShift CLI:

curl -LO https://mirror.openshift.com/pub/openshift-v4/clients/oc/latest/linux/oc.tar.gz
tar -xvf oc.tar.gz
sudo mv oc /usr/local/bin/
oc version

# 3.3 Instalar o OpenShift Cluster (OKD)
Baixe o OpenShift e extraia os arquivos:
curl -LO https://github.com/openshift/okd/releases/latest/download/openshift-install-linux.tar.gz
tar -xvf openshift-install-linux.tar.gz
sudo mv openshift-install /usr/local/bin/
# Crie um diretório de configuração:
mkdir ~/openshift
cd ~/openshift

# Crie o arquivo de configuração install-config.yaml:
apiVersion: v1
baseDomain: example.com
metadata:
  name: openshift
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 2
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 1
networking:
  networkType: OpenShiftSDN
platform:
  none: {}
pullSecret: '{"auths": {}}'

# Inicie a instalação:
openshift-install create cluster --dir=~/openshift

# 4. Criando a Máquina para Ansible
Repita os passos de criação da VM (AWS, Azure, GCP), mas use uma máquina menor:

CPU: 4 vCPUs
RAM: 16GB
Disco: 50GB SSD
Conecte-se à máquina e instale o Ansible:

sudo dnf install -y ansible
ansible --version
Instale os módulos Kubernetes:
ansible-galaxy collection install kubernetes.core

# 5. Configurando OpenShift no Ansible
Agora, vamos configurar a integração do OpenShift com o Ansible.

# Crie um Service Account no OpenShift:
oc create sa ansible-sa -n openshift
oc adm policy add-cluster-role-to-user cluster-admin -z ansible-sa -n openshift
# Recupere o token:
oc serviceaccounts get-token ansible-sa -n openshift

# 6. Criando Playbook do Ansible para Automação
Crie o arquivo inventory.yaml:

all:
  children:
    openshift:
      hosts:
        cluster.openshift.local:
          ansible_host: "https://api.openshift.local:6443"
          ansible_user: "kubeadmin"
          ansible_password: "<SENHA_DO_OPENSHIFT>"
          ansible_python_interpreter: /usr/bin/python3
          
# Crie o playbook.yaml:

- name: Criar um projeto no OpenShift
  hosts: localhost
  gather_facts: no
  tasks:
    - name: Criar namespace
      kubernetes.core.k8s:
        state: present
        kind: Namespace
        name: meu-projeto
# Execute o playbook:
ansible-playbook -i inventory.yaml playbook.yaml

# Verifique no OpenShift:
oc get namespaces
