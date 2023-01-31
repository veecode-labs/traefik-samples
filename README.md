# traefik-samples

Exemplos de Traefik para Meetups.

- [traefik-samples](#traefik-samples)
- [Pré-requisitos](#pré-requisitos)
  - [Docker Engine local (Docker Desktop ou similar)](#docker-engine-local-docker-desktop-ou-similar)
  - [Docker Engine em nuvem (DigitalOcean)](#docker-engine-em-nuvem-digitalocean)
  - [Cluster Kubernetes em nuvem (DigitalOcean)](#cluster-kubernetes-em-nuvem-digitalocean)
- [Traefik Simples (Local)](#traefik-simples-local)
- [Traefik na DigitalOcean (Remoto)](#traefik-na-digitalocean-remoto)
- [Traefik no Kubernetes](#traefik-no-kubernetes)
  - [Instalando o Traefik via Helm Chart](#instalando-o-traefik-via-helm-chart)
  - [Instalando uma aplicação web e seu IngressRoute](#instalando-uma-aplicação-web-e-seu-ingressroute)


# Pré-requisitos

## Docker Engine local (Docker Desktop ou similar)

Qualquer engine de containers local é suficiente.

## Docker Engine em nuvem (DigitalOcean)

Este lab espera que você tenha criado uma VM na DigitalOcean com um Docker Engine já instalado. Isto pode ser feito facilmente de duas formas:

1. Pela UI web da DigitalOcean, criando um droplet a partir da imagem "Docker 20.10.21 on Ubuntu" ou similar
2. Usando a linha de comando:

```sh
doctl compute droplet create traefik-docker --region nyc1 --image docker-20-04 --size s-1vcpu-1gb --ssh-keys "sua_fingerprint_ssh"
```

Anote o IP público desta VM. O acesso a esta VM poderá ser feito via SSH:

```sh
ssh -i path_para_sua_chave root@seu_ip_publico
```

DICA: abra as portas 80 e 443 caso não estejam aberts:

```sh
ufw allow http
ufw allow https
ufw status verbose
```

Você pode criar a seguinte entrada em `~/.ssh/config` para facilitar o uso desta VM:

```ssh-config
Host docker-traefik
  HostName seu_ip_publico
  IdentityFile "~/.ssh/id_do"
  User root
  ServerAliveInterval 60
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
```

Note que isto permite acessar o Docker Engine via SSH. Isto pode ser feito de forma "bruta", via DOCKER_HOST:

```sh
export DOCKER_HOST=ssh://docker-traefik
docker version
```

...ou usando contexto:

```
docker context create \
  --docker "host=ssh://docker-traefik" \
  --description "My remote docker engine" \
  docker-traefik
docker context use docker-traefik
```

## Cluster Kubernetes em nuvem (DigitalOcean)

Um script `create-k8s.sh` foi fornecido para criar um cluster kubernetes na DigitalOcean, mas você também pode criá-lo diretamente com os comandos abaixo:

```sh
doctl kubernetes cluster create meetup --region nyc3 --node-pool "name=mainpool;size=s-2vcpu-4gb;count=3"
doctl kubernetes cluster kubeconfig save meetup
# para testar
kubectl get nodes
```

# Traefik Simples (Local)

```sh
docker context use default
cd docker
docker-compose up
```

Teste as aplicações app1 e app2 acessando as mesmas via Traefik:

```sh
curl app1.localhost
curl app2.localhost
# ou, sem depender de /etc/hosts
curl -H "Host: app1.localhost" 127.0.0.1
curl -H "Host: app2.localhost" 127.0.0.1
```

As rotas foram configuradas pela inspeção do Docker Engine feita pelo próprio Traefik:

```yaml
  webapp1:
    image: containous/whoami
    labels:
      - "traefik.http.routers.webapp1.rule=Host(`app1.localhost`)"
    command:
      - "--name=app1"
```

Opcional: o dashboard do Traefik está disponível em http://localhost:8080 .

# Traefik na DigitalOcean (Remoto)

Os nomes DNS `app1.labs.vee.codes` e `app2.labs.vee.codes` já foram previamente ajustados para o IP público da VM.

```sh
docker context use docker-traefik
cd docker
docker-compose -f docker-compose-cloud.yml up
```

Notas:

- O uso do "docker context" faz com que o `docker-compose` controle o engine remoto via SSH
- Estamos usando servidores de staging da Let's Encrypt (comente a linha `caserver` para usar produção)
- Preciso remover comentários para as apps usarem HTTPS
- Inicialmente o tráfego HTTPS ocorre usando um certificado auto-assinado "TRAEFIK DEFAULT CERT" (que não é útil)
- Após remover comentários para as apps usarem HTTPS o certificado será gerado via Let's Encrypt ("STAGING")
- Após comentar a linha `caserver` o certificado gerado será válido

Para inspecionar o certificado:

```sh
curl -k -v https://app1.labs.vee.codes
```

# Traefik no Kubernetes

## Instalando o Traefik via Helm Chart

O Traefik pode ser instalado via helm chart:

```sh
# importar o chart do Traefik
helm repo add traefik https://traefik.github.io/charts
helm repo update
# instalar o Traefik usando o chart
helm upgrade -i traefik traefik/traefik -f k8s/values-traefik.yaml
```

O `values-traefik.yaml` utilizado é trivial (apenas aponta a versão beta do Traefik e pede persistência de dados para os certificados Let's Encrypt):

```yaml
image:
  name: traefik
  # v3.0 ainda em beta
  tag: "v3.0"
persistence:
  enabled: true
```

O Traefik será instalado como **ingress controller** do cluster (tipo `LoadBalancer`), recebendo portanto um IP público que será obtido dinamicamente. Você deve aguardar até que este IP seja gerado:

```sh
kubectl get svc -w
...
NAME         TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)                      AGE
traefik      LoadBalancer   10.245.220.205   159.203.148.35   80:30286/TCP,443:30984/TCP   2m2s
```

**Nota:** você pode criar entradas DNS manualmente para este IP ou pode usar soluções como o external-dns, mas não iremos abordar isto aqui. Estamos assumindo que uma entrada DNS `whoami.labs.vee.codes` já foi ajustada para este IP (159.203.148.35).

Observe no painel da DO que o cluster foi criado e que o load balancer com o IP público também está disponível, roteando requisições HTTP e HTTPS para o Traefik. Como não há nenhuma aplicação no cluster ainda, qualquer requisição para as portas 80 e 443 dará erro 404 (Not Found), o que é comportamento esperado para um ingress controller.

## Instalando uma aplicação web e seu IngressRoute

Vamos usar a mesma aplicação de exemplo de Docker ("whoami") mas vamos instalar com seu helm chart:

```sh
# importar o chart (única vez)
helm repo add cowboysysop https://cowboysysop.github.io/charts/
helm repo update
# instalar whoami
helm upgrade -i whoami cowboysysop/whoami -f k8s/values-whoami.yaml
```

O `values-whoami.yaml` utilizado omite o objeto Ingress do Kubernetes, pois usaremos uma configuração do Traefik:

```yaml
ingress:
  enabled: false
```

A configuração específica para o Traefik expor a aplicação via HTTPS com geração de certificados delegada ao Let's Encrypt será feita por um objeto IngressRoute, instalado com o comando abaixo:

```sh
kubectl apply -f k8s/ingress_route.yml
```

Com esta configuração o próprio Traefik será capaz de rotear o tráfego e negociar a geração e renovação de certificados com o Let's Encrypt.

A aplicação está disponível via HTTPS:

```sh
# https com certificado inválido (STAGING)
curl -k -v https://whoami.labs.vee.codes/
```

**Nota:** para que o Let's Encrypt gere um certificado válido será necessário comentar a entrada "caserver" no arquivo `values-traefik.yaml` e reinstalar o Traefik:

```yaml
additionalArguments:
...
  # - "--certificatesresolvers.letsencrypt.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory"
```
