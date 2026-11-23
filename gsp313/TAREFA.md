# GSP313 - Implementação do Cloud Load Balancing no Compute Engine: Laboratório com Desafio

> **Plataforma:** Google Cloud Skills Boost (Google Skills)  
> **Código do Laboratório:** `GSP313`

---

## Visão Geral e Cenário do Desafio

Você atua como engenheiro de nuvem júnior responsável por fornecer funcionalidade de rede para instâncias de máquina virtual (VM) do Compute Engine em uma rede de nuvem privada virtual (VPC) do Google Cloud.

Para oferecer suporte ao balanceamento de carga do tráfego de rede, é necessário compreender a diferença entre um **Balanceador de Carga de Rede (Network Load Balancer)** e um **Balanceador de Carga HTTP (HTTP Load Balancer)** e configurar ambos para aplicações executadas em VMs do Compute Engine.

### Objetivos Principais
- Criar várias instâncias de servidor Web com regras de firewall.
- Configurar um serviço de balanceamento de carga de rede.
- Criar um balanceador de carga HTTP com Grupo Gerenciado de Instâncias (MIG).

### Padrões Obrigatórios
- **Região padrão:** `us-east1`
- **Zona padrão:** `us-east1-c`
- **Rede:** `default`

---

## Tarefa 1: Criar várias instâncias do servidor da Web

Criar três instâncias de VM do Compute Engine na rede `default`, instalar o servidor Web Apache em cada uma e configurar a regra de firewall `www-firewall-network-lb` permitindo tráfego HTTP (porta `80`).

### Especificações das VMs

| Propriedade | Valor |
| :--- | :--- |
| **Instância de VM - 1** | `web1` |
| **Instância de VM - 2** | `web2` |
| **Instância de VM - 3** | `web3` |
| **Região** | `us-east1` |
| **Zona** | `us-east1-c` |
| **Série** | `E2` |
| **Tipo de máquina** | `e2-small` |
| **Tags de rede** | `network-lb-tag` |
| **image-family** | `debian-12` |
| **image-project** | `debian-cloud` |
| **Regra de firewall** | `www-firewall-network-lb` (TCP:80) |

### Script de Inicialização (Startup Script)
*(Atualizando `web-number` para `web1`, `web2` ou `web3` conforme o nome da instância)*

```bash
#!/bin/bash
apt-get update
apt-get install apache2 -y
service apache2 restart
echo "<h3>Servidor da web: web-number</h3>" | tee /var/www/html/index.html
```

---

## Tarefa 2: Configurar o serviço de balanceamento de carga (Network Load Balancer)

Criar os recursos de suporte ao balanceamento de carga regional de rede e enviar tráfego para a regra de encaminhamento para validar a distribuição entre as VMs.

### Especificações

| Propriedade | Valor |
| :--- | :--- |
| **IP externo estático** | `network-lb-ip-1` |
| **Target-pool** | `www-pool` |
| **Portas** | `80` |

---

## Tarefa 3: Criar um balanceador de carga HTTP

Configurar um balanceador de carga HTTP global utilizando um Grupo Gerenciado de Instâncias (MIG), verificação de integridade, mapa de URL e proxy HTTP de destino.

### Especificações

| Propriedade | Valor |
| :--- | :--- |
| **Modelo de back-end (Instance Template)** | `lb-backend-template` |
| **Tags** | `allow-health-check` |
| **Grupo gerenciado de instâncias (MIG)** | `lb-backend-group` |
| **Tipo de máquina (`machine-type`)** | `e2-medium` |
| **Imagem (`image-family` / `image-project`)** | `debian-12` / `debian-cloud` |
| **Regra de firewall** | `fw-allow-health-check` |
| **Intervalos de origem (Firewall)** | `130.211.0.0/22`, `35.191.0.0/16` |
| **Tráfego / Porta** | Entrada (`INGRESS`) / `80` |
| **Endereço IP externo** | `lb-ipv4-1` |
| **Verificação de integridade** | `http-basic-check` |
| **Mapa de URL** | `web-map-http` |
| **Proxy HTTP de destino** | `http-lb-proxy` |

---

## Validação Final

1. No Console do Google Cloud, acessar **Serviços de rede > Balanceamento de carga**.
2. Selecionar o balanceador `web-map-http`.
3. Testar o acesso via navegador ou `curl` em `http://[IP_ADDRESS]/` usando o IP externo reservado (`lb-ipv4-1`) e confirmar a entrega da página pelas instâncias do grupo gerenciado.