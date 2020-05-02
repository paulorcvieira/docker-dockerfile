# Dockerfile

## Conceitos
[Doc - DockerFile](https://docs.docker.com/engine/reference/builder/)
[Doc - Formato do Dockerfile](https://docs.docker.com/engine/reference/builder/#format)

FROM Informa a partir de qual imagem ser'a gerada a nova imagem

[Doc - FROM](https://docs.docker.com/engine/reference/builder/#from)

[Doc - LABEL](https://docs.docker.com/engine/reference/builder/#label)

[Doc - Docker Build](https://docs.docker.com/engine/reference/commandline/build)

Para rodar o Dockerfile, entramos na pasta e rodamos o comando:
```bash
$ docker build .
```
Perceba que o *.* segnifica que o arquivo está na pasta onde esta sendo rodado o comando.

## A instrução RUN
[Doc - RUN](https://docs.docker.com/engine/reference/builder/#run)
- Padrão: RUN ["executable", "param1", "param2"] (exec form)

Para criar uma imagem modificada rodamos o comando:
```bash
$ docker build -t <repository>:version
```

Como exemplo de uma nova camada baseada na imagem ubuntu, com o comando RUN do docker file para instalar o zip
```bash
$ docker build -t ubuntu:latest .
```
######  Dockerfile
```
FROM ubuntu:latest
LABEL maintainer="paulorcvieira"

RUN apt-get update && apt-get install -y zip
```

## As instruções CMD e EXPOSE

### EXPOSE
Vamos criar uma nova imagem dessa vez para o serviço Ubuntu SSH
[Docker Hub - Ubuntu SSH](https://hub.docker.com/r/rastasheep/ubuntu-sshd/)

[Doc - EXPOSE](https://docs.docker.com/engine/reference/builder/#expose)

Para deixar uma porta específica aberta em uma imagem usamos o comando EXPOSE <porta>

###### Dockerfile
```bash
FROM  ubuntu:latest
LABEL maintainer="paulorcvieira"

# Atualiza o SO
RUN apt-get update
RUN apt-get install -y openssh-server vim curl
RUN mkdir /var/run/sshd

# Configura o SSH
RUN echo 'root:root' |chpasswd
RUN sed -ri 's/^PermitRootLogin\s+.*/PermitRootLogin yes/' /etc/ssh/sshd_config
RUN sed -ri 's/UsePAM yes/#UsePAM yes/g' /etc/ssh/sshd_config

EXPOSE 22
EXPOSE 3030

CMD    ["/usr/sbin/sshd", "-D"]
```

Perceba que usamos o comando CMD

Vamos testar
```bash
$ docker build -t rastasheep/ubuntu-sshd  .
docker container run -d -P rastasheep/ubuntu-sshd
docker port <container id>
ssh root@localhost -p <porta>
```

E já estamos dentro de nossa máquina

Caso tenha problema ao iniciar o ssh podemos apagar a chave antiga, para isso basta rodar o comando:
```bash
rm -rf <caminho>
```

### CMD
[Doc - CMD](https://docs.docker.com/engine/reference/builder/#cmd)
Serve basicamente para executar um comando, no caso acima foi utilizado para deixar o servidor ativo em foreground


## As instruções ADD e COPY

[Doc - COPY](https://docs.docker.com/engine/reference/builder/#copy)

- Padrão: COPY [--chown=<user>:<group>] ["<src>",... "<dest>"]

Com o COPY faz uma copia dos arquivos ou diretórios para dentro da imagem


[Doc - ADD](https://docs.docker.com/engine/reference/builder/#add)

- Padrão: ADD [--chown=<user>:<group>] ["<src>",... "<dest>"]

Com o ADD podemos adicionar um arquivo que estaja fora da imagem para dentro da imagem, diferenciando-se na possibilidade de adicionar arquivos que estão em determinadas URLs para dentro da imagem, e também adicionar arquivos *.tar.gz* já descomprimidos para dentro da imagem.

Vamos criar um exemplo, e colocar uma menssagem de boas-vindas a nossa imagem Ubuntu-SSH

[SSH Message](https://www.daveperrett.com/articles/2007/03/27/change-the-ssh-login-message/)

```bash
$ docker container run -d -P <image ssh>
$ docker container port <container id>
$ ssh root@localhost -p <port>
```

Agora que estamos dentro da imagem, vamos precisar atualizar e instalar o *vim*

```bash
$ apt-get update && apt-get install -y vim
```

Agora precisamos editar o arquivo *sshd_config* no caminho `vim /etc/ssh/sshd_config` e descomentar a linha *Banner /etc/banner*, mas vamos fazer isso diretamente no Dockerfile

Vamos adicionar essa linha no final do arquivo diretamente pelo Dockerfile, para isso vamos criar uma pasta e um arquivo `etc/banner` e dentro desse arquivo banner vamo colocar a nossa tela de boas-vindas

```bash
              |
.  .    . .-. | .-..-. .--.--. .-.
 \  \  / (.-' |(  (   )|  |  |(.-'
  `' `'   `--'` `-'`-' '  '  ` `--'
```

Nosso dockerfile ficou assim

```bash
FROM  ubuntu:latest
LABEL maintainer="paulorcvieira"

# Atualiza o SO
RUN apt-get update
RUN apt-get install -y openssh-server vim curl
RUN mkdir /var/run/sshd

# Configura o SSH
RUN echo 'root:root' |chpasswd
RUN sed -ri 's/^PermitRootLogin\s+.*/PermitRootLogin yes/' /etc/ssh/sshd_config
RUN sed -ri 's/UsePAM yes/#UsePAM yes/g' /etc/ssh/sshd_config

# Boas-Vindas
RUN echo 'Banner /etc/banner' >> /etc/ssh/sshd_config
COPY etc/banner /etc/

# Expõe as portas
EXPOSE 22
EXPOSE 3030

CMD    ["/usr/sbin/sshd", "-D"]
```

Então podemos rodar o build no nosso Dockerfile

```bash
$ docker images
$ docker build -t paulorcvieira/ubuntu-sshd-banner:latest .
```

Feito isso vamos parar nosso container

```bash
$ docker container ls
$ docker container stop <container id>
$ docker container prune
```

Agora vamos renomear nossa nova imagem

```bash
$ docker images
$ docker image tag <image id> username_dockerhub/image_name:versao
```

No meu caso ficou assim:
```bash
$ docker image tag ecd0ecded855 paulorcvieira/ubuntu-sshd-banner:latest
```

E vamos levantar nossa imagem e acessar o ssh

```bash
$ docker container run -d -P paulorcvieira/ubuntu-sshd-banner
$ docker container port <container id>
```

## A Instrução USER
[Doc - USER](https://docs.docker.com/engine/reference/builder/#user)

Agora vamos criar um usuario para instalar alguns pacotes, para isso vamos criar o usuário em nosso Dockerfile

```bash
RUN useradd -ms /bin/bash app
RUN adduser app sudo
RUN echo 'app:app' |chpasswd
```

Já podemos instalar o NVM com usuário *app* com o comando USER, e vamos aproveitar e instalar a última versão estável do node

```bash
USER app
RUN /bin/bash -l -c "curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.35.3/install.sh | bash"
RUN /bin/bash -l -c ". ~/.nvm/nvm.sh && nvm 12.16.3"
```

Agora precisamos voltar para o usuário root para usar o EXPOSE

```bash
USER ROOT
EXPOSE 22
EXPOSE 3030
```

Então nosso Dockerfile deve ter ficado assim

```bash
FROM  ubuntu:latest
LABEL maintainer="paulorcvieira"

# Atualiza o SO
RUN apt-get update
RUN apt-get install -y openssh-server vim curl
RUN mkdir /var/run/sshd

# Configura o SSH
RUN echo 'root:root' |chpasswd
RUN sed -ri 's/^PermitRootLogin\s+.*/PermitRootLogin yes/' /etc/ssh/sshd_config
RUN sed -ri 's/UsePAM yes/#UsePAM yes/g' /etc/ssh/sshd_config

# Boas-Vindas
RUN echo 'Banner /etc/banner' >> /etc/ssh/sshd_config
COPY etc/banner /etc/

# Adciona o usuário 'app'
RUN useradd -ms /bin/bash app
RUN adduser app sudo
RUN echo 'app:app' |chpasswd

# Altera para o usuário 'app'
USER app

# Instala o NVM
RUN /bin/bash -l -c "curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.35.3/install.sh | bash"
RUN /bin/bash -l -c ". ~/.nvm/nvm.sh && nvm install 12.16.3"

# Altera para o usuário 'root'
USER root

# Expõe as portas
EXPOSE 22
EXPOSE 3030

CMD    ["/usr/sbin/sshd", "-D"]
```

Agora já podemos dar um build em nosso Dockerfile

```bash
$ docker build -t paulorcvieira/ubuntu-sshd:latest .
```

Podemos levantar nossa imagem com o usuário *app* e senha *app*

```bash
$ docker container run -d -P <image id>
$ docker container port <container id>
$ ssh app@localhost -p <port>
$ nvm list
$ exit
```

## A instrução VOLUME
[Doc - VOLUME](https://docs.docker.com/engine/reference/builder/#volume)

Aqui o docker faz o mapeamento para o computador caso não tenhamos iniciado um container com a instrução *$(pwd)* para indicar o local de armazenamento de nossos dados

```bash
FROM  ubuntu:latest
LABEL maintainer="paulorcvieira"

# Atualiza o SO
RUN apt-get update
RUN apt-get install -y openssh-server vim curl
RUN mkdir /var/run/sshd

# Configura o SSH
RUN echo 'root:root' |chpasswd
RUN sed -ri 's/^PermitRootLogin\s+.*/PermitRootLogin yes/' /etc/ssh/sshd_config
RUN sed -ri 's/UsePAM yes/#UsePAM yes/g' /etc/ssh/sshd_config

# Boas-Vindas
RUN echo 'Banner /etc/banner' >> /etc/ssh/sshd_config
COPY etc/banner /etc/

# Adciona o usuário 'app'
RUN useradd -ms /bin/bash app
RUN adduser app sudo
RUN echo 'app:app' |chpasswd

# Altera para o usuário 'app'
USER app

# Instala o NVM
RUN /bin/bash -l -c "curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.35.3/install.sh | bash"
RUN /bin/bash -l -c ". ~/.nvm/nvm.sh && nvm install 12.16.3"

# Altera para o usuário 'root'
USER root

# Expõe as portas
EXPOSE 22
EXPOSE 3030

# Cria e configura o ponto de montagem do volume
RUN mkdir /workspace
RUN chmod 777 /workspace
VOLUME /workspace

CMD    ["/usr/sbin/sshd", "-D"]
```

Já podemos rodar o docker build

```bash
$ docker build -t paulorcvieira/ubuntu-ssh:latest .
```

Agora vamos acessar o ssh sem o *$(pwd)* e o docker se encarrega de no utilizar o volume que indicamos no Dockerfile, isso sem a necessidade de utilizarmos o `$ docker container run -d -P -v $(pwd):/workspace <image id>`
```bash
$ docker images
$ docker container run -d -P <image id>
```

## Docker Hub
[Doc - Push-Images](https://docs.docker.com/datacenter/dtr/2.2/guides/user/manage-images/pull-and-push-images/)

Agora vamos subir nossa imagem para o *Docker Hub* para isso vamos renomear nossa imagem

```bash
$ docker image tag <image id> dockerhub_username/image_name:latest
$ docker image tag ecd0ecded855 paulorcvieira/ubuntu-sshd-banner:latest
```

Agora precisamos fazer o login em nossa conta do Docker Hub

```bash
$ docker login
```

Agora realemente vamos enviar nossa imagem para o Docker Hub

```bash
$ docker push paulorcvieira/ubuntu-sshd-nvm:latest
```

Agora está imagem já esta disponível para download

```bash
$ docker pull paulorcvieira/ubuntu-sshd-nvm
```

# Pronto!