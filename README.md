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

### Primeiramente, vamos criar um diretório para o arquivo YAML e mover para ele:
```
    apt  install docker.io
    mkdir Graylog-Docker-compose
    cd Graylog-Docker-compose
    apt install git 
    git clone git@github.com:Reisvmr/Graylog-Docker-compose.git
    docker-compose up -d --build
    
```
-----------------
# Configuração de encaminhamento RSyslog

## Visão geral
 Procurando centralizar o registro para nossa equipe de desenvolvimento no Graylog via Logstash. 
 Portanto, estou abordando as coisas como uma chance de entender o RSyslog e seus recursos como remetente de logs.

## Procedimento

### Configurar escuta de TCP no host de índice de log

Remova o comentário das seguintes linhas em `/etc/rsyslog.conf`. Isso permitirá que o daemon rsyslog escute as solicitações de entrada na porta TCP514. Estamos usando o TCP aqui para que possamos ter alguma confiança de que as mensagens dos hosts do agente alcançam o indexador. (Mais sobre isso abaixo)

```
# Fornece recepção de syslog TCP
$ModLoad imtcp
$InputTCPServerRun 514
```

Adicione uma linha a `/etc/rsyslog.conf` para realmente colocar os logs recebidos em um arquivo específico.

```
local3.* /local/logs/httpd-error
local4.* /local/logs/httpd-access
```

Finalmente, reinicie o processo rsyslog.

`` `bash
 services rsyslog restart
`` `

### Configure o host do agente

No host do agente, o host que está executando o apache, adicione um arquivo, `/etc/rsyslog.d/apache.conf`. Isso será lido na hora de início do syslog. Este arquivo diz ao rsyslog para ler `/var/log/httpd/error_log` (o log de erro padrão do apache no CentOS) ou `/var/log/apache2/error.log` (o log de erro padrão do apache no Ubuntu/Debian)a cada 10 segundos e enviar suas mensagens para o recurso` local3.info` no syslog. (Expandido para também ler logs de acesso e enviá-los para `local4.info`)


```
$ModLoad imfile

# Registro de erro padrão do Apache
$InputFileName /var/log/apache2/error.log # (o log de erro padrão do apache no Ubuntu/Debian)
#$InputFileName /var/log/httpd/error_log # (o log de erro padrão do apache no CentOS)
$InputFileTag httpd-error-default:
$InputFileStateFile stat-httpd-error
$InputFileSeverity info
$InputFileFacility local3
$InputRunFileMonitor

# Log de acesso padrão do Apache
$InputFileName /var/log/apache2/access.log # (o log de erro padrão do apache no Ubuntu/Debian)
# $InputFileName /var/log/httpd/access_log #  (o log de erro padrão do apache no CentOS)
$InputFileTag httpd-access-default:
$InputFileStateFile stat-httpd-access
$InputFileSeverity info
$InputFileFacility local4
$InputRunFileMonitor
$InputFilePollInterval 10

```

Em seguida, modifique `/etc/rsyslog.conf`, descomente ou adicione as seguintes linhas ao final do arquivo. Isso diz ao rsyslog para configurar uma fila de log e encaminhar qualquer mensagem de log de instalação `local3` e` local4` para a porta TCP X.X.X.X. `@@` é uma abreviação de rsyslog para a porta syslog TCP. Se você deseja encaminhar via UDP, use um único `@` ao invés. No entanto, provavelmente não vale a pena configurar a fila nesse caso, pois o rsyslog não tem uma maneira de garantir que os pacotes UDP sejam recebidos pelo host de índice.

Nesta configuração, o host do agente armazenará todas as mensagens de log que não podem ser enviadas ao host de índice. Isso é bom para lidar com momentos em que o host de índice está sendo reinicializado ou, de outra forma, indisponível.

```
$ WorkDirectory /var/lib/ rsyslog # onde colocar os arquivos de spool
$ ActionQueueFileName fwdRule1 # prefixo de nome exclusivo para arquivos de spool
$ ActionQueueMaxDiskSpace 1g # 1gb limite de espaço (use o máximo possível)
$ ActionQueueSaveOnShutdown ao # salvar mensagens no disco ao desligar
$ ActionQueueType LinkedList # executado de forma assíncrona
$ ActionResumeRetryCount -1 # tentativas infinitas se o host estiver inativo
local3. * @@10.32.208.70:PORTA
local4. * @@10.32.208.70:PORTA
```
Por último, reinicie rsyslog no host do agente.

```bash
service httpd restart
```
### Teste a configuração

Agora, se você reiniciar o servidor apache (`service httpd restart`), você deve ver os logs sendo gerados em`/local/logs/httpd-error` no host de índice. Caso contrário, verifique se há algum bloqueio de firewall entre os hosts e se as alterações de configuração do rsyslog estão sendo analisadas corretamente. Você pode verificar a configuração do rsyslog com este comando: `/sbin/rsyslogd -c5 -f /etc/rsyslog.conf -N1`

### Configure o Logstash para consumir os novos arquivos

Essa configuração usa um analisador grok personalizado para extrair o nível de erro da mensagem, bem como extrair a tag que definimos na configuração do rsyslog para seu próprio campo. Para obter ajuda com filtros de grok personalizados, verifique http://grokdebug.herokuapp.com/

```
input {
  file {
    type => "httpd-error-log"
    path => ["/local/logs/httpd-error"]
    sincedb_path => "/opt/logstash/sincedb-access"
    discover_interval => 10
  }

  file {
    type => "httpd-access-log"
    path => ["/local/logs/httpd-access"]
    sincedb_path => "/opt/logstash/sincedb-access"
    discover_interval => 10
  }
}

filter {
  if [type] == "httpd-error-log" {
    grok {
      match       => [ "message", "\S+ \d+ \d+:\d+:\d+ %{HOSTNAME} %{NOTSPACE:tag}: \[%{DAY} %{MONTH} %{MONTHDAY} %{TIME} %{YEAR}\] \[%{LOGLEVEL:level}\] %{GREEDYDATA}" ]
    }
    mutate {
      rename      => [ "program", "vhost" ]
    }
  }

  if [type] == "httpd-access-log" {
    grok {
      match       => [ "message", "\S+ \d+ \d+:\d+:\d+ %{HOSTNAME} %{NOTSPACE:tag}: %{COMBINEDAPACHELOG}" ]
      add_field		=> { "level", "info" }
    }
  }
}

output {
  elasticsearch {
    host => "localhost"
  }
}

```
