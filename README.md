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


![digAnswer](https://github.com/filipeborges/bindDnsConfig/blob/master/Assets/testRequestWithDig.png)

Neste caso estamos fazendo uma consulta ao host de nome *__www__* dentro da zona *__exemplo.com__*;

# Atualizando uma zona e finalizando o NAMED

* Caso se deseje atualizar os registros de uma zona:
- Inserir novos registros no arquivo com os registros da zona  e *__realizar o incremento do valor numérico no registro SOA (primeiro valor)__*. No caso do servidor configurado, o arquivo com os registros da zona seria o *__exemplo.com.zone__*;
- Uma vez que o arquivo foi atualizado, deve-se mandar o sinal SIGHUP ao processo do NAMED, para que o arquivo seja carregado novamente:
```
sudo kill -SIGHUP {pid_named}
```

* Caso se deseje finalizar o servidor de nomes NAMED:
```
sudo kill -SIGTERM {pid_named}
```

# Configurando acesso ao NAMED via RNDC

O RNDC é um utilitário de controle do NAMED que pode ser usado tanto de forma local quanto remota. Com ele é possível adicionar uma zona sem precisar mexer diretamente no arquivo de zonas, recarregar o arquivo de configuração sem o envio de sinal SIGHUP, finalizar o NAMED sem o envio de sinal SIGTERM, consultar o status do servidor e vários outros comandos úteis na administração de um servidor DNS. Para maiores informações sobre o RNDC, execute:
```
man rndc
```

* Toda a comunicação entre NAMED (Servidor) e RNDC (Cliente) precisa ser autenticada com uma chave. Portanto para configurar o acesso ao NAMED via RNDC, primeiro precisamos gerar essa chave de autenticação usando o seguinte comando:
```
dnssec-keygen -a hmac-md5 -b 256 -n HOST my_key
```
Serão gerados dois arquivos com prefixos *__Kmy_key__*, onde um arquivo será um .key e o outro um .private. Obtenha o valor da linha *__Key:__* através do comando:
```
cat nome_arquivo.private
```

* Uma vez que a chave foi gerada, vamos criar o arquivo *__rndc.key__* no diretório /etc/bind/ conforme o exemplo abaixo:

```
key "rndc_key" {
        algorithm hmac-md5;
        secret "{valor_chave}";
};
```
OBS: O token {valor_chave} deve ser substituído pela chave obtida com o comando CAT anteriormente. Uma boa prática é colocar restrições no acesso de leitura e de escrita a este arquivo, de forma que somente usuários autorizados possam visualizar e modificar a chave de autenticação.

* Agora vamos editar o arquivo *__named.conf__* acrescentando no início do arquivo o código:
```
include "/etc/bind/rndc.key";

controls {
        inet * allow { localhost; }
        keys {rndc_key;};
};

```
A linha *__inet * allow { localhost; }__* indica para o NAMED aceitar conexões vindas do *__localhost__* em todos os IPs locais. A linha *__keys {rndc_key;};__* indica para usar a chave definida no arquivo rndc.key com o nome "rndc_key". Caso fosse necessário um acesso remoto, o parametro *__allow__* poderia ser configurado como:
```
inet * allow { localhost; {ip_host_remoto}; }
```

* Por fim devemos criar o arquivo *__rndc.conf__* no diretório /etc/bind/, conforme o exemplo a seguir:
```
options {
        default-server  localhost;
        default-key     "rndc_key";
};

key "rndc_key" {
        algorithm hmac-md5;
        secret "{valor_chave}";
};
```
OBS: {valor_chave} deve ser a mesma chave configurada no arquivo *__rndc.key__*. Caso o acesso fosse feito remotamente, este arquivo deveria ser configurado na máquina de acesso, e a linha *__default-server__* deveria conter o IP do servidor rodando o NAMED. É uma boa prática restringir o acesso à leitura e a escrita a este arquivo, de forma a proteger a chave de autenticação.

* Por fim devemos enviar um comando ao NAMED via RNDC da seguinte forma:
```
rndc -c /etc/bind/rndc.conf {comando}
```

A lista de comandos possíveis pode ser visualizada através do comando:
```
man rndc
```

# Configurando uma Zona Reversa

A zona reversa é o inverso da zona convencional do DNS, ou seja, mapeia um IP de um host para um nome de domínio. Para mapear os IPs de hosts, dentro de uma rede, para nomes de domínio, a rede deve estar no formato xxx.xxx.xxx.xxx/24 (IP de rede formado por 24 bits). Considere a zona reversa a seguir:
```
zone "1.1.10.in-addr.arpa" {
        type master;
        file "/etc/named_test/exemplo.com.rr.zone";
};
```
Nesta zona reversa, podemos mapear todos os hosts da rede 10.1.1.x/24 para seus respecitvos nomes de domínio. Note que somente os octetos do IP relacionados a rede é que vão declarados na zona, de forma invertida (1.1.10 = 10.1.1.x/24), e fazendo parte do domínio *__in-addr.arpa__*, que é um domínio especial usado somente em zonas reversas. Se queremos mapear reversamente todos os hosts da rede 192.168.0.x/24, devemos declarar o nome da zona como: *__0.168.192.in-addr.arpa__*. A declaração do nome da zona, conforme o exemplo acima, deve ser feita no arquivo *__named.conf__*.

Maiores informações sobre a zona reversa em ![DNS Reverse Mapping](http://www.zytrax.com/books/dns/ch3/)

* Considerando o arquivo *__named.conf__* anteriormente configurado no DNS convencional, ao adicionarmos a zona reversa para a rede *__10.1.1.x/24__*, supondo que tivessemos autoridade sobre este conjunto de IPs, obtemos a seguinte configuração:
```
/*Zona configurada no processo de DNS convencional*/
zone "myzone.com" {
        type master;
        file "/etc/named_test/exemplo.com.zone";
};
/*Zona reversa adicionada*/
zone "1.1.10.in-addr.arpa" {
        type master;
        file "/etc/named_test/exemplo.com.rr.zone";
};
```

* O próximo passo é criar o arquivo da zona reversa *__exemplo.com.rr.zone__* no diretório /etc/bind/, conforme o exemplo abaixo:
```
@           IN  SOA localhost.  localhost.localhost.com   (
    1   ; Valor numerico que deve ser incrementado a cada vez que este arquivo for alterado para indicar ao NAMED para recarregar o arquivo.
    1D  ; Tempo que o servidor secundario deve esperar até perguntar ao servidor primario se houve alteração no arquivo da zona.
    10M ; Tempo que o servidor secundario ira esperar pela resposta do servidor primario, ate reenviar uma nova requisicao.
    10M ; Tempo que o servidor secundario ira esperar pela resposta do reenvio de requisicao ao servidor primario. Caso este tempo estoure, o servidor secundario para de responder como autoridade para a zona.
    1D )  ; Tempo TTL para cache das informacoes da zona por outros servidores de nome.
                IN      NS      localhost.

5               IN      PTR     host1.exemplo.com.
6               IN      PTR     host2.exemplo.com.
```

Os registros de recurso SOA e NS são os mesmos que no caso da zona convencional. A diferença aqui se dá por conta do registro PTR, que é usado para apontar (pointer) um nome de domínio para um determinado IP. Portanto, ao consultarmos no servidor de DNS por informações reversa dos IPs *__10.1.1.5__* e *__10.1.1.6__*, será retornado respectivamente os nomes de domínio *__host1.exemplo.com__* e *__host2.exemplo.com__*.

* Podemos testar a zona reversa usando a ferramenta *__dig__* da seguinte forma:
```
dig @localhost -x {IP}
```
