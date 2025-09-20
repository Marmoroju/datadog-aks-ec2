# Datadog - Monitoramento e Observabilidade

O [Datadog](https://www.datadoghq.com/) tem como objetivo:
- Unificar o monitoramento de aplicações, infraestrutura e logs
- Detectar problemas de performance em tempo real
- Gerar Alertas automáticos com base em métricas e eventos
- Ajudar no troubleshooting rápido com dashboards interativos
- Facilitar observabilidade em ambientes cloud e microsserviços
- Promover visibilidade de ponta a ponta do ambiente DevOps

Os três pilares da Observabilidade:
- Métricas
    - Números que mostram o estado e desempenho da sua aplicação ou infraestrutura longo do tempo. Exemplo: uso de CPU, memória, requisições por segundo.

- Logs
    - Registros detalhados de eventos que aconteceram no sistema. Exemplo: erros, acessos, mensagens do app.

- Traces
    - Rastreamento de ponta a ponta de uma requisição passando por vários serviços. Ajuda a entender onde está o gargalo ou erro em um fluxo complexo.


## Instalação do agente do Datadog no AKS

### Passo 01 - Configuração do ambiente Azure
Criar Resource Group
```bash
az group create --name aks-free-rg-mmrj --location eastus
```
Criar cluster - Esse processo é demorado. 
```bash
az aks create \
--resource-group aks-free-rg-mmrj \
--name aks-free-cluster \
--node-count 1 \
--enable-addons monitoring \
--generate-ssh-keys \
--enable-managed-identity \
--tier free \
--location eastus
```
Acessar o cluster AKS
```bash
az aks get-credentials --resource-group aks-free-rg-mmrj --name aks-free-cluster --overwrite-existing
```
Criar Namespace para infra do datadog
```bash
kubectl create namespace datadog
```
### Passo 02 - Instalação do Helm
- [HELM](https://helm.sh/docs/intro/quickstart/)

```bash
sudo snap install helm --classic
```

```bash
helm version
```

### Passo 03 - Adicionar e configurar repositório do Datadog
1. Crie sua conta em [Datadog](https://www.datadoghq.com/)
2. Configure sua Stack
3. Escolha uma plataforma para instalar os agentes - `Kubernetes`
    - AKS
4. Adicione o repositório Helm Chart. Copie, cole e execute os comandos já sugeridos, com um detalhe no `create secret` que será preciso adicioná-lo ao namespace criado.
```bash
helm repo add datadog https://helm.datadoghq.com
helm repo update
kubectl create secret generic datadog-secret --from-literal api-key=<SUA-API-KEY> -n datadog
```
5. Customize your observability coverage. Cada objeto selecionado será adicionado ao arquivo datadog-values.yaml logo em seguida.
6. Como arquivo criado, executar o comando abaixo para instalar/desinstalar o agente.

Instalar:
```bash
helm install datadog-agent -n datadog -f datadog-values.yaml datadog/datadog
```
Verificar Pods com o agente instalado:
```bash
kubectl get pods -n datadog
```
Desinstalar o agente:
```bash
helm uninstall datadog-agent -n datadog
```
8. Na página do datadog, clique em `Finish` para ter acesso ao dashboard. Neste momento será iniciada a coleta de métricas do seu cluster que pode ser conferida no menu lateral `Infrastructure`

## Instalação do agente do Datadog no EC2

### Passo 01 - Configuração do ambiente na AWS 
Security Group
```bash
aws ec2 create-security-group \
--group-name datadog-sg \
--description "Security group para agente do Datadog"
```
Liberar porta 22 para acesso SSH
```bash
aws ec2 authorize-security-group-ingress \
--group-name datadog-sg \
--protocol tcp \
--port 22 \
--cidr 0.0.0.0/0
```
Liberar porta 443 para acessar a aplicação
```bash
aws ec2 authorize-security-group-ingress \
--group-name datadog-sg \
--protocol tcp \
--port 443 \
--cidr 0.0.0.0/0
```
Criar KEY_PAIR
```bash
aws ec2 create-key-pair --key-name "datadog" --query 'KeyMaterial' --output text > "datadog.pem"
```
Dar Permissão para a KEY_PAIR
```bash
chmod 400 "datadog.pem"
```
Criar instância EC2 na região us-east-1. AMI Ubuntu
```bash
aws ec2 run-instances \
--image-id ami-0360c520857e3138f \
--instance-type t2.micro \
--key-name datadog \
--security-groups datadog-sg \
--tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=datadog-ec2}]' \
--count 1
```
Recuperar nome e IP da instância EC2
```bash
aws ec2 describe-instances \
--filters "Name=tag:Name,Values=datadog-ec2" \
--query "Reservations[*].Instances[*].[InstanceId,PublicIpAddress]" \
--output table
```
Acessar a instância via SSH adicionando o IP recuperado. O nome de usuário depende da AMI (imagem) usada.
- Amazon Linux = ec2-user
- Ubuntu = ubuntu
- CentOS = centos
```bash
ssh -i "datadog.pem" ubuntu@3.80.99.92
```

### Passo 02 - Instalar o agente do Datadog na instância EC2

1. Acessar o painel do Datadog
2. Menu lateral > Integrations:
    - Fleet Automation
    - Install agents
    - Host Based > Linux
3. 'Run the install command'
    - Select API Key
    - Use API Key
    - Copie todos os comandos, cole na instância EC2 e execute
4. Execute os comandos sugeridos no `Output` gerados no final da instalação.
5. Acesse o menu lateral > `Infrastructure` e acompanhe. Leva um certo período de tempo para começar a coletar as métricas.

## Criar uma aplicação no cluser AKS e monitorrar pelo Datadog
Criar namespace
```bash
kubectl create namespace java-api
```
Deploy da aplicação
```bash
kubectl apply -f deployment.yml -n java-api
```
Verificar o IP gerado pelo Service
```bash
kubectl get svc -n java-api
```

## Traces e Logs 
- Logs
    - Acesse o menu lateral > `clique duas vezes` em Logs
    - Painel de Log Explorer
    - Pesquise `service:java-api `
- Traces
    - Acesse o menu lateral > APM
    - Escolha a apliação java-api
    - Menu Superior > Traces
    - Ao selecionar qualquer linha do trace, ele abrirá um painel com a linha do tempo daquela execução, com isso você pode acompanhar em que momento ela levou mais tempo para carregar.

## Dashboards
Acesse o menu lateral > Dashboards. Nele existe uma grande quantidade de dashboards já disponíveis para uso, basta escolher o que melhor se encaixa para sua coleta de métricas e observabildiade.

Caso contrário, você pode criar um do `praticamente zero`, isso mesmo, você pode Criar um novo Dashboard e em uma outra aba abrir um modelo pronto que lhe interessa, copiar uma parte dele e colar no seu `novo dashboard`

## Monitores
Responsáveis por gerar alertas. Você configura um limite de métrica e ela é ultrapassada, lhe enviado uma mensagem pelo slack ou outro canal.

Menu lateral > Monitors > + New Monitor
- Escolha o tipo de métrica e alerta
- Monte de acordo com suas especificações
- 1 - Escolha o método de detecção
- 2 - Defina a Métrica > `kubernetes.cpu.usage.total` > from cluster_name:aks-free-cluster env:dev
- 3 - Defina a condição de alerta
    - Alert threshold > 0.5
    - Warning threshold > 0.3
- 4 - Configure a mensagem de alerta automatizada que você irá receber.
    - Título do alerta 
    - Corpo da Mensagem do Alerta
    - pressione `@` para selecionar para qual canal configurado será enviado o alerta
    - Teste a notificação antes de Criar
    - `Crie` o alerta se a mensagem chegar conforme definido

Integrar o Monitor com sua conta na AWS, Azure ou outra. Ele irá monitorar toda sua conta.
- Menu lateral > Integrations > Integrations
- AWS
- Escolha o método - Cloudformation > Terraform > Manually

## Lembrete
Excluir recursos criados na Azure e AWS

### AWS
Interromper Instancia
```bash
aws ec2 stop-instances --instance-ids <INSTANCE_ID> 
```
Terminar a Instancia
```bash
aws ec2 terminate-instances --instance-ids <INSTANCE_ID> 
```
Filtrar ID Security Group
```bash
aws ec2 describe-security-groups \
  --filters Name=group-name,Values=datadog-sg \
  --query "SecurityGroups[*].GroupId" \
  --output text
```
Apagar o Security Group
```bash
aws ec2 delete-security-group --group-id <SG_ID>
```
Apagar a KEY_PAIR local
```bash
rm <KEY_NAME>.pem"
```
### Azure
Cluster AKS
```bash
az aks delete --name aks-free-cluster --resource-group aks-free-rg-mmrj --yes --no-wait
```
Resource Group
```bash
az group delete --name aks-free-rg-mmrj --yes --no-wait
```

 Sempre confira em cada uma das Clouds os recursos foram removidos.

