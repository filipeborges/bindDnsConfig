# Instalação BIND 9

O processo de instalação do BIND focará no sistema operacional Ubuntu, porém este processo provavelmente pode ser aplicado a sistemas operacionais que fazem parte da família Debian. A versão do Bind utilizada foi a 9.10.4-P4, portanto este guia focará nesta versão.

Existem dois métodos de instalação do BIND:
1. Compilar e instalar diretamente do código fonte.
2. Instalação via gerenciador de pacote do próprio sistema operacional.

### 1. Instalando diretamente do código fonte
**Requisitos:** Sistema UNIX, compilador C ANSI.

1. Extraia o conteúdo do arquivo *__bind-9.10.4-P4.tar.gz__* e acesse o diretório raíz.
2. Configure os arquivos necessários para a build do BIND executando: 
```
./configure [--prefix={dir_instalacao}]
```
OBS: Por padrão os arquivos serão instalados no diretório **/usr/local/bin** caso o parametro **--prefix** não seja informado. **dir_instalacao** deve ser um diretório absoluto.
3. Para instalar, execute:
```
make install
```

### 2- Instalando via gerenciador de pacotes (APT):
```
sudo apt-get update
sudo apt-get install bind9 bind9utils
```
