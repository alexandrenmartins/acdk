### Bem Vindo ao Treinamento de Docker

Neste treinamento aprenderemos a utilizar funções básicas do Docker, como por exemplo:

* Gerenciar, Baixar e Executar Imagens
* Expor portas e Utilizar Variáveis
* Utilizar Volumes
* Buildar e subir Para um Registry

Vamos lá !!!

---

### Obtendo do Ambiente

```
docker info
```

### Verificando o que está rodando

```
docker ps
```

### Verificando as imagens baixadas

```
docker images
```

---

### Baixando uma imagem
Imagem do [nginx](https://hub.docker.com/_/nginx):

```
docker pull nginx
```

### Executando uma imagem

```
docker run nginx
```

* O Comando "run" por si só já efetua o pull caso a imagem não exista no host.

### Porque o shell ficou preso?
Bom "Ctrl+C" para sair! Para responder esta questão, vamos antes executar a aplicação "httping", que é uma ferramenta simples para teste de URLs.

```
docker run bretfisher/httping
```

Repare que foi retornado a versão da ferramenta e logo largou o shell.

```
docker run bretfisher/httping google.com
```

Agora o comando predendeu o shell em execução, isso deve-se por conta de como o ENTRYPOINT e o CMD do Dockerfile da imagem foi definido.

### ENTRYPOINT vs CMD
Em um Dockerfile, podemos ter um ou outro ou ambos.

Vamos primeiro ver o [Dockerfile](https://github.com/nginxinc/docker-nginx/blob/master/mainline/debian/Dockerfile) do Nginx, procure no final do arquivo por ENTRYPOINT, veja que o aquivo [docker-entrypoint.sh](https://github.com/nginxinc/docker-nginx/blob/master/mainline/debian/docker-entrypoint.sh) é referenciado e no final dele temos '"exec "$@"' que absorve todos os parâmetros do CMD do Dockerfile, que é em si a chamada da daemom do Nginx Server que de fato prendeu o shell.

Agora vamos fazer o mesmo com o [Dockerfile](https://github.com/BretFisher/httping-docker/blob/main/Dockerfile) do Httping, repare que o ENTRYPOINT é um comando Shell direto do executável da aplicação, httping -Y, em seguida o CMD com o parâmentro --version. Ao executar o Docker run com o parâmentro google.com ao final, na verdade estamos sobrepondo o CMD do Dockerfile, sendo assim, podemos perceber que:

* ENTRYPOINT: Estático
* CMD: Parametrizável

---


### Baixando uma imagem
Imagem do [ubuntu](https://hub.docker.com/_/ubuntu):

```
docker pull ubuntu

docker run ubuntu
```

### Porque não prendeu o shell?
Ao observar o [Dockerfile](https://github.com/tianon/docker-brew-ubuntu-core/blob/f2f3f01ed67bab2e24b8c4fda60ef035a871b4c7/focal/Dockerfile) do Ubuntu, percebemos que só existe um CMD executando o "bash", ou seja não é um Daemon, podemos então prender este shell, alocando um TTY a ele:

```
docker run -i -t ubuntu
```

Parâmetro  | Descrição
---------- | -----------
-i | Mantem o STDIN
-t | Aloca um pseudo TTY

Ao executar o comando, somos transferidos para dentro do shell do container, e estamos isolados do sistema operacional hospedeiros, e ao sair, o container entra em estado de stop. para sair digite `exit` e de enter.

Para ver o container em estado de stop é necessário adicionar o parâmetro "-a":

```
docker ps -a
```

Aproveite para observar um nome bem criativo gerado automaticamente na última coluna para o container.

#### Outros parâmentos do docker run:

Parâmetro  | Descrição
---------- | -----------
--rm | Remove o container quando sai
--restart | Política de restart: always / on-failure:n / unless-stopped
--name | Personaliza o nome para o container em execução
-d | Roda como daemon em background
-p | Publica a porta do container

### Rodando o Nginx como Daemon

Devemos observar o seguinte, para saber em que porta o container roda, devemos sempre consultar a [documentacao da imagem](https://hub.docker.com/_/nginx), logo perceberemos que ela roda na porta 80, sendo assim vamos mapear a imagem para rodar na porta 8080 do host com o parâmetro "-p" e o "-d" para rodar em background:

```
docker run --name nginx -d -p 8080:80 nginx
```

### Vamos observar no Sistema operacional o que aconteceu:

```
netstat -tupan | grep LISTEN
```

### Testando a aplicação

```
curl 127.0.0.1:8080
```

ou pelo link do [host]({{TRAFFIC_HOST1_8080}})

### Vamos parar o container:

```
docker stop nginx
```

### Agora tente rodar novamente:

```
docker run --name nginx -d -p 8080:80 nginx
```

Repare que agora recebemos uma mensagem de conflito pois o container com o nome "nginx" já existe. Desta vez como nomeamos não podemos sobrepor, antes eram nomes aleatórios gerados pelo docker.

### Para ver o container em stop rode:

```
docker ps -a
```

### Para remover rode:

```
docker rm nginx
```

Caso deseje apenas iniciar novamente o container, execute:

```docker start nginx```

### Mudando a porta

```
docker run --rm --name nginx -d -p 80:80 nginx
```

### Testando a aplicação na nova porta

```
curl 127.0.0.1:80
```

ou pelo link do [host]({{TRAFFIC_HOST1_80}})

---


### Extraindo o Log de um container
```
docker logs nginx --tail 10 -f
```

### Entrando em um container em execução
```
docker exec -it nginx bash
```

### Inspecionando a rede

### Observer a interface de bridge criada pelo docker:

```
docker network ls
```

### No Json podemos ver os endereços IPs de overlay da rede interna do docker para cada container:

```
docker network inspect bridge | jq
```

### No sistema operacional: 

```
ip a
```

### Vamos parar novamente o container:

```
docker stop nginx
```

Repare que desta vez com o parâmetro "--rm", ao dar stop, o docker automaticamente removeu o container.

```
docker ps -a
```

---


#### Passando variáveis para o container
Vamos criar dois containers com a imagem de teste "simple-webapp-color" observe no comando o parâmetro "-e APP_COLOR=", respectivamente irão rodar dois containers rosa e azul nas portas 8080 e 8081 para não ocorrer conflito de portas no hospedeiro:

```
docker run --rm --name fundo_rosa -e APP_COLOR=pink -d -p 8080:8080 mmumshad/simple-webapp-color
```

Veja a app [fundo_rosa]({{TRAFFIC_HOST1_8080}})

```
docker run --rm --name fundo_azul -e APP_COLOR=blue -d -p 8081:8080 mmumshad/simple-webapp-color
```

Veja a app [fundo_azul]({{TRAFFIC_HOST1_8081}})

#### Persistindo dados por volumes
Como já sabemos, containers são efêmeros, não persistem dados caso sejam removidos, para isso podemos mapear volumes no host ou até remotamente.

### Dentro do diretório verifique o seguinte arquivo:

```
cat ~/volume/index.html
```

### Mapeando o Volume

O Parâmetro "-v" irá fazer a ligação do volume local com o diretório dentro do container (vide documentação da imagem):

```
docker run --rm --name nginx -d -p 80:80 -v ~/volume:/usr/share/nginx/html nginx
```

Experimente alterar o arquivo no hospedeiro e visualizar a [aplicação]({{TRAFFIC_HOST1_80}})

### Inspecione o ambiente e limpe

```
docker network inspect bridge | jq

docker ps

docker stop fundo_rosa

docker stop fundo_azul

docker stop nginx

docker ps
```

---


### Criando uma imagem

Vamos agora criar uma aplicação bem simples em python utilizando o Flask, esta aplicação posuirá um único arquivo chamado "app.py" e seu Dockerfile com as instruções de build.

### A aplicação app.py

```
cat ~/build/app.py
```

### O Dockerfile

```
cat ~/build/Dockerfile
```

### Iniciando o processo de build

```
docker build -t flask-app ~/build/.
```

### Verificando a imagem gerada

```
docker images
```

### Retagiando a Imagem

```
docker tag flask-app flask-app:v1
```

### Rodando a Imagem
```
docker run --rm --name flask-app -d -p 9090:8080 flask-app:v1
```

### Testando a aplicação

```
curl 127.0.0.1:9090
```

ou pelo link do [host]({{TRAFFIC_HOST1_9090}})

---


### Instanciando um Docker Registry

Vamos agora criar um docker registry local utilizando a [imagem oficial](https://hub.docker.com/_/registry) do docker, o registry possui inúmeras [configurações](https://docs.docker.com/registry/configuration/), como por exemplo autenticação via token, persistencia em bucket S3, aqui iremos configurar tudo de maneira simples apenas para provar o conceito.

### Vamos instanciar o Docker Registry na porta padrão 5000:

```
docker run -d -p 5000:5000 --restart always --name registry registry:2
```

### Verificando o Container

```
docker ps
```

### Verificando a imagem da App

```
docker images
```

### Retagiando a imagem com o endereço do Registry

```
docker tag flask-app:v1 localhost:5000/flask-app:v1
```

### Verificando a imagem da App

```
docker images
```

### Fazendo o Upload da Imagem para o Registry

```
docker push localhost:5000/flask-app:v1
```

### Consultando o Registry

```
curl http://localhost:5000/v2/_catalog | jq

curl http://localhost:5000/v2/flask-app/tags/list | jq
```

### Baixando a Imagem

```
docker pull localhost:5000/flask-app:v1
```

* É importande perceber que quando executamos um pull (download) de uma imagem sem qualquer endereço explícito ou tag como por exemplo:

```
docker pull nginx
```

* O Docker primeiro irá procura-la localmente, se não encontra-la irá em hub.docker.com, seu repositório padrão, caso não especificada, a tag será latest.

---

### Assets

```
mkdir -p build

mkdir -p volume



cat > build/app.py <<EOF
import os
from flask import Flask
app = Flask(__name__)

@app.route("/")
def main():
    return "Bem Vindo ao Docker Build!"

@app.route('/comovai')
def hello():
    return 'Muito Bem!'

if __name__ == "__main__":
    app.run()
EOF


cat > build/Dockerfile <<EOF
FROM ubuntu:18.04

RUN apt-get update && apt-get install -y python python-pip

RUN pip install flask 

COPY app.py /opt/

USER 1001

ENTRYPOINT FLASK_APP=/opt/app.py flask run --host=0.0.0.0 --port=8080
EOF


cat > volume/index.html <<EOF
<html>
<h1> Testando Volume ... </h1>
</html>
EOF
```

```
cat > /etc/docker/daemon.json <<EOF
{
  "insecure-registries" : ["http://$(hostname -s):5000"]
}
EOF

systemctl restart docker
```