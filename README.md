# Instalação BIND 9

O processo de instalação do BIND focará no sistema operacional Ubuntu, porém este processo provavelmente pode ser aplicado a sistemas operacionais que fazem parte da família Debian. A versão do Bind utilizada foi a *__9.10.4-P4__*, portanto este guia focará nesta versão.

Existem dois métodos de instalação do BIND:

1. Compilar e instalar diretamente do código fonte.
2. Instalação via gerenciador de pacote do próprio sistema operacional.

### 1. Instalando diretamente do código fonte
**Requisitos:** Sistema UNIX, compilador C ANSI.

- Extraia o conteúdo do arquivo *__bind-9.10.4-P4.tar.gz__* e acesse o diretório raíz.
- Configure os arquivos necessários para a build do BIND executando: 
```
./configure [--prefix={dir_instalacao}]
```
OBS: Por padrão os arquivos serão instalados no diretório **/usr/local/bin** caso o parametro **--prefix** não seja informado. **dir_instalacao** deve ser um diretório absoluto.

- Para instalar, execute:
```
make install
```

### 2- Instalando via gerenciador de pacotes (APT):
```
sudo apt-get update
sudo apt-get install bind9 bind9utils
```

# Subindo um servidor de DNS simples

Uma vez que o BIND se encontra instalado na máquina o próximo passo é realizar a sua configuração. O servidor de DNS do BIND se chama **NAMED** e seu arquivo de configuração é o *__named.conf__*. Este arquivo pode ser criado do zero ou então utilizar o arquivo default que vem junto com a instação do BIND (/etc/bind/named.conf). Neste guia configuraremos um servidor primário simples, criando um arquivo do zero.

- Crie o arquivo *__named.conf__* conforme o exemplo abaixo, no diretório  /etc/bind/. Maiores informações sobre as opções e sintaxe deste arquivo podem ser consultadas em: [16.2. /etc/named.conf](https://www.centos.org/docs/5/html/Deployment_Guide-en-US/s1-bind-namedconf.html)

```
/*Configurando a zona ficticia exemplo.com*/
zone "exemplo.com" {
    type master; /* Servidor primario */
    file "/etc/bind/exemplo.com.zone";  /*Arquivo com registros de recurso da zona*/
};
```

- Crie o arquivo *__exemplo.com.zone__* conforme o exemplo abaixo, no diretório /etc/bind/. Maiores informações sobre as opções e sintaxe deste arquivo podem ser consultadas em: [16.3. Zone Files](https://www.centos.org/docs/5/html/Deployment_Guide-en-US/s1-bind-zone.html) 

```
@   		IN  SOA localhost.  localhost.localhost.com   (
    1   ; Valor numerico que deve ser incrementado a cada vez que este arquivo for alterado para indicar ao NAMED para recarregar o arquivo.
    1D  ; Tempo que o servidor secundario deve esperar até perguntar ao servidor primario se houve alteração no arquivo da zona.
    10M ; Tempo que o servidor secundario ira esperar pela resposta do servidor primario, ate reenviar uma nova requisicao.
    10M ; Tempo que o servidor secundario ira esperar pela resposta do reenvio de requisicao ao servidor primario. Caso este tempo estoure, o servidor secundario para de responder como autoridade para a zona.
    1D )  ; Tempo TTL para cache das informacoes da zona por outros servidores de nome.

@   		IN  NS  localhost. ; Registro para o servidor de nome autoritativo para a zona (O nome do servidor deve estar no formato Full Qualified Domain Name).
localhost   IN  A   127.0.0.1 ; Registro mapeando o hostname do servidor de nome da zona para seu IP.
www 		IN  A   192.168.17.22 ; Registro mapeando um servidor HTTP ficticio para seu respectivo IP.
```

- Inicie o servidor executando:
```
sudo named -c /etc/bind/named.conf
```
OBS: Por padrão o NAMED roda na porta UDP 53, que é a porta padrão de serviços DNS. Por está razão, abrir um socket UDP nesta porta necessita de privilégios de super usuário.

# Testando o servidor DNS configurado

* O servidor configurado pode ser testado usando a ferramenta *__Dig__*, da seguinte forma:

```
dig @localhost www.exemplo.com 
```
Neste caso estamos fazendo uma consulta ao host de nome *__www__* dentro da zona *__exemplo.com__*;

# Atualizando uma zona e finalizando o NAMED

* Caso se deseje atualizar os registros de uma zona:
1. Inserir novos registros no arquivo com os registros da zona  e *__realizar o incremento do valor numérico no registro SOA (primeiro valor)__*. No caso do servidor configurado, o arquivo com os registros da zona seria o *__exemplo.com.zone__*;
2. Uma vez que o arquivo foi atualizado, deve-se mandar o sinal SIGHUP ao processo do NAMED, para que o arquivo seja carregado novamente:
```
sudo kill -SIGHUP {pid_named}
```

* Caso se deseje finalizar o servidor de nomes NAMED:
```
sudo kill -SIGTERM {pid_named}
```
