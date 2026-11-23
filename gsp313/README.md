# Solução Completa - GSP313: Implementação do Cloud Load Balancing no Compute Engine

Este documento reúne todos os comandos `gcloud` testados e validados para concluir o laboratório **GSP313** com pontuação máxima (`100/100`), além das lições aprendidas e armadilhas evitadas durante a execução.

---

## Arquitetura Implementada

1. **Camada de Instâncias Individuais + Network Load Balancer (Regional `us-east1`):**
   - 3 VMs (`web1`, `web2`, `web3`) na zona `us-east1-c` rodando Apache2.
   - Firewall `www-firewall-network-lb` liberando TCP/80 para a tag `network-lb-tag`.
   - IP estático regional `network-lb-ip-1`, `http-health-check` legado (`basic-check`), `target-pool` (`www-pool`) e regra de encaminhamento `www-rule`.

2. **Camada Escalável + HTTP(S) Load Balancer (Global):**
   - Template `lb-backend-template` (`e2-medium`, `debian-12`, tag `allow-health-check`).
   - Grupo Gerenciado de Instâncias `lb-backend-group` em `us-east1-c` com porta nomeada `http:80`.
   - Firewall `fw-allow-health-check` permitindo sondas dos ranges do Google Cloud (`130.211.0.0/22` e `35.191.0.0/16`).
   - IP global `lb-ipv4-1`, `health-check` `http-basic-check`, `backend-service` `web-backend-service`, `url-map` `web-map-http`, proxy `http-lb-proxy` e regra global `http-content-rule`.

---

## Passo a Passo da Execução (Cloud Shell)

### Tarefa 1: Criar as VMs do Servidor Web e o Firewall

> **Nota técnica:** Geramos os arquivos de startup script localmente com `cat << EOF` antes de criar as instâncias para evitar expansão de histórico do Bash (`!`) e garantir que o metadado `startup-script` contenha o nome literal de cada VM (`web1`, `web2`, `web3`).

```bash
# 1. Gerar os scripts de inicialização individuais
for i in 1 2 3; do
  cat << EOF > startup-web${i}.sh
#!/bin/bash
apt-get update
apt-get install apache2 -y
service apache2 restart
echo "<h3>Servidor da web: web${i}</h3>" | tee /var/www/html/index.html
EOF
done

# 2. Criar as instâncias web1, web2 e web3
for i in 1 2 3; do
  gcloud compute instances create web${i} \
    --zone=us-east1-c \
    --machine-type=e2-small \
    --network=default \
    --tags=network-lb-tag \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --metadata-from-file=startup-script=startup-web${i}.sh
done

# 3. Criar a regra de firewall para tráfego HTTP
gcloud compute firewall-rules create www-firewall-network-lb \
  --network=default \
  --allow=tcp:80 \
  --target-tags=network-lb-tag
```

#### Validação da Tarefa 1
```bash
gcloud compute instances list --filter="name:(web1 web2 web3)"
# Testar cada IP externo retornado:
# curl http://[IP_EXTERNO_VM]
```

---

### Tarefa 2: Configurar o Balanceador de Carga de Rede (Network LB)

> **Ponto crítico para aprovação:** O `target-pool` (`www-pool`) exige uma verificação de integridade HTTP legada (`gcloud compute http-health-checks`) associada a ele para que o avaliador automático reconheça a configuração como completa.

```bash
# 1. Reservar o IP externo estático regional
gcloud compute addresses create network-lb-ip-1 \
  --region=us-east1

# 2. Criar a verificação de integridade HTTP legada
gcloud compute http-health-checks create basic-check

# 3. Criar o target pool vinculado ao health check
gcloud compute target-pools create www-pool \
  --region=us-east1 \
  --http-health-check=basic-check

# 4. Adicionar as instâncias ao pool
gcloud compute target-pools add-instances www-pool \
  --region=us-east1 \
  --instances=web1,web2,web3 \
  --instances-zone=us-east1-c

# 5. Criar a regra de encaminhamento na porta 80
gcloud compute forwarding-rules create www-rule \
  --region=us-east1 \
  --ports=80 \
  --address=network-lb-ip-1 \
  --target-pool=www-pool
```

#### Validação da Tarefa 2
```bash
IPADDRESS=$(gcloud compute addresses describe network-lb-ip-1 --region=us-east1 --format="get(address)")
for n in $(seq 1 10); do curl -s http://$IPADDRESS; echo ""; done
```

---

### Tarefa 3: Criar o Balanceador de Carga HTTP (Global)

```bash
# 1. Criar o script de startup para o template do MIG
cat << 'EOF' > lb-startup.sh
#!/bin/bash
apt-get update
apt-get install apache2 -y
service apache2 restart
echo "<h3>Servidor da web: $(hostname)</h3>" | tee /var/www/html/index.html
EOF

# 2. Criar o modelo de instância (Instance Template)
gcloud compute instance-templates create lb-backend-template \
  --region=us-east1 \
  --machine-type=e2-medium \
  --network=default \
  --tags=allow-health-check \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --metadata-from-file=startup-script=lb-startup.sh

# 3. Criar o Grupo Gerenciado de Instâncias (MIG)
gcloud compute instance-groups managed create lb-backend-group \
  --base-instance-name=lb-backend \
  --size=2 \
  --template=lb-backend-template \
  --zone=us-east1-c

# 4. Definir a porta nomeada 'http' no grupo de instâncias
gcloud compute instance-groups managed set-named-ports lb-backend-group \
  --zone=us-east1-c \
  --named-ports=http:80

# 5. Criar a regra de firewall para os Health Checks do Google Cloud
gcloud compute firewall-rules create fw-allow-health-check \
  --network=default \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:80 \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=allow-health-check

# 6. Reservar o IP externo global
gcloud compute addresses create lb-ipv4-1 \
  --ip-version=IPV4 \
  --global

# 7. Criar o Health Check HTTP
gcloud compute health-checks create http http-basic-check \
  --port=80

# 8. Criar o Backend Service e anexar o MIG
gcloud compute backend-services create web-backend-service \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=http-basic-check \
  --global

gcloud compute backend-services add-backend web-backend-service \
  --instance-group=lb-backend-group \
  --instance-group-zone=us-east1-c \
  --global

# 9. Criar o Mapa de URL, Proxy HTTP e Regra de Encaminhamento Global
gcloud compute url-maps create web-map-http \
  --default-service=web-backend-service

gcloud compute target-http-proxies create http-lb-proxy \
  --url-map=web-map-http

gcloud compute forwarding-rules create http-content-rule \
  --address=lb-ipv4-1 \
  --global \
  --target-http-proxy=http-lb-proxy \
  --ports=80
```

#### Validação da Tarefa 3
```bash
# Verificar se os backends estão HEALTHY
gcloud compute backend-services get-health web-backend-service --global

# Testar o IP global do balanceador HTTP
LB_IP=$(gcloud compute addresses describe lb-ipv4-1 --global --format="get(address)")
curl -i http://$LB_IP
```

---

## Resumo de Lições Aprendidas (Troubleshooting)

1. **Formatação no enunciado:** No texto original da Tarefa 1, os comandos `service apache2 restart` e `echo ...` apareciam unidos sem quebra de linha (`restartecho`). Separar em linhas distintas evita falha na execução do script.
2. **Metadados exatos na Tarefa 1:** O script de validação do laboratório verifica o conteúdo literal de `startup-script` em cada VM (`web1`, `web2`, `web3`).
3. **Health Check legado na Tarefa 2:** Mesmo sem estar explícito na tabela resumida da Tarefa 2, o `www-pool` precisa de um `http-health-check` (`basic-check`) vinculado para passar na avaliação automática.
4. **Propagação do HTTP Load Balancer na Tarefa 3:** O balanceador global pode levar de 2 a 5 minutos para ficar `HEALTHY` e parar de retornar `Connection reset by peer` após a criação da `forwarding-rule`.