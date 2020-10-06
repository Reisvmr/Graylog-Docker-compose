# Graylog-Docker-compose
Criação de arquivo docker compose para o Graylog
-----

### Passo 1 — Como instalar o Docker Compose

Embora possamos instalar o Docker Compose a partir dos repositórios oficiais do Ubuntu, ele está várias versões menores atrás do último lançamento, então vamos instalar o Docker Compose do repositório do GitHub do Docker. O comando abaixo é ligeiramente diferente daquele que você encontrará na página dos Lançamentos. Use a flag -o para especificar o arquivo de saída primeiro ao invés de redirecionar a saída, essa sintaxe evita executar um erro de autorização negada causada ao usar o sudo.

Vamos verificar o lançamento atual e, se necessário, atualizá lo no comando abaixo:
```
    sudo curl -L https://github.com/docker/compose/releases/download/1.21.2/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
```
### Em seguida, vamos definir as permissões:
```
    sudo chmod +x /usr/local/bin/docker-compose
```
### Então, vamos verificar se a instalação foi bem sucedida verificando a versão:
```
    docker-compose --version
```
### Isso aparecerá na tela a versão que instalamos:

```
Output
docker-compose version 1.21.2, build a133471
```
Agora que temos o Docker Compose instalado, estamos prontos para executar um exemplo “Hello World”.
Passo 2 — Como executar um Contêiner com o Docker Compose

O registro público do Docker, o Docker Hub, inclui uma imagem do Hello World para demonstração e teste. Ele ilustra a configuração mínima necessária para executar um contêiner utilizando o Docker Compose: um arquivo YAML que chama uma única imagem:

Primeiramente, vamos criar um diretório para o arquivo YAML e mover para ele:
```
    mkdir hello-world
    cd hello-world
```
