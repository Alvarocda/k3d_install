# Comandos de instalacao K3D

Site do K3D com tutorial de instalação:
https://k3d.io/v5.4.6/


## Comando para criação do cluster
```docker
k3d cluster create -p "9080:80@loadbalancer" -p "9443:443@loadbalancer" --agents 2 --k3s-arg "--no-deploy=traefik@server:*" --agents-memory 2048MiB cluster-teste
```


## Instalar o rancher
```docker
docker run --privileged -d --restart=unless-stopped -p 8080:80 -p 8443:443 rancher/rancher
```


## Instalar HAProxy
```bash
apt-get install haproxy
```



## Instalar o NGinx Ingress Controller
A instalação do controller foi feita usando a opção charts do rancher.

 - Procure pelo chart NGINX Ingress Controller e clique em instalar
 - Marcar a caixa "Customize Helm Options before install"
 - Em settings, na opção Installation Kind, deixar marcado como daemonSet
 - Em Service o Kind deve ser do tipo LoadBalancer, a porta HTTP e HTTPS devem ser as mesmas usadas na criação do cluster 9080 e 9443
 - na Aba Prometheus desativar o serviço (Opcional)

Após fazer essas configuração, NÃO avance de etapa ainda, na parte superior existe um botão chamado "Edit Yaml", ao clicar nessa opção, você vera o YAML do helm que sera instalado no servidor.
Procure por "httpsPort", ela vai estar mapeada da seguinte maneira no YAML:
```yaml
httpsPort:
  enable: true
  nodePort: ''
  port: 9443
  targetPort: 443
```

Mude o targetPort para 80, sim, tanto o http quanto o https apontaram para a porta 80 do container.

Vai ficar dessa maneira:
```yaml
httpsPort:
  enable: true
  nodePort: ''
  port: 9443
  targetPort: 80
```

Após isso, siga com a instalação normal do controller.


## Configuração do HAPROXY

Agora precisamos configurar o haproxy para, para que funcione corretamente, precisamos direcionar todas as requsiçoes HTTP e HTTPS que vem de conexões externas para o Ingress, mas antes precisamos saber qual é o IP do ingress, para isso, digite o seguinte comando

kubectl get svc -n [namespace-do-ingress]

Ao digitar esse comando, o output no console deve ser algo parecido com isso:  

```
NAME                                     TYPE           CLUSTER-IP      EXTERNAL-IP                        PORT(S)                         AGE
nginx-ingress-controller-nginx-ingress   LoadBalancer   10.43.21.78     172.30.0.2,172.30.0.3,172.30.0.4   9080:32251/TCP,9443:32678/TCP   68m

```

Anote todos os IPs que estão marcados como EXTERNAL IP

Agora, edite o arquivo de configuração do haproxy em
 /etc/haproxy/haproxy.cfg

 ```yaml
 global
        log /dev/log    local0
        log /dev/log    local1 notice
        chroot /var/lib/haproxy
        stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
        stats timeout 30s
        user haproxy
        group haproxy
        daemon


defaults
        log     global
        mode    tcp
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http

frontend main
    bind :80
    mode tcp
    default_backend rancher80

frontend main443
    bind :443 ssl crt /opt/alvarocda.dev/alvarocda.dev.pem crt /opt/wildcard.alvarocda.dev/wildcard.alvarocda.dev.pem
    mode tcp
    default_backend rancher80


backend rancher80
    mode tcp
    server worker1 172.30.0.2:9080
    server worker2 172.30.0.3:9080
    server worker3 172.30.0.4:9080
backend rancher443
    mode tcp
    server worker1 172.30.0.2:9443
    server worker2 172.30.0.3:9443
    server worker3 172.30.0.4:9443

```

Aqui você deve substituir todos os modes de http para tcp.
Embaixo de mode nas opções defaults tem um log (não lembro o nome do que ta escrito, mas pode remover também).

Caso queira usar um certificado SSL, use o seguinte comando para criar um unico arquivo pem usando o arquivo de chave privada e fullchain

```bash
cat privkey.pem fullchain.pem > chave_unificada.pem
```


feito isso, salve o arquivo de configuração do haproxy e reinicie o serviço utilizando o seguinte comando.

```bash
systemctl restart haproxy
```


## Criando o ingress
Na hora de criar o ingress, navegue até a aba "Ingress Class" e selecione nginx, após fazer isso, vai começar a funcionar corretamente o ingress