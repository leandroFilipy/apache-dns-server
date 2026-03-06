# Apache HTTP Server

### O que é?
O Apache é um projeto de servidor estático desenvolvido pela Apache. É um projeto que visa desenvolver e manter um servidor HTTP Open-Source para sistemas operacionais modernos, como os UNIX e Windows.

Foi lançado em 1995 e se popularizou em 1996, dominando o mercado de servidores HTTP até meados de 2004, quando o Nginx foi considerado superior.

### Diferenças entre Server Estático e Dinâmico
Os servidores estáticos não mudam, eles são transmitidos e nada em seu conteúdo muda conforme o tempo.

Os servidores dinâmicos mudam conforme o tempo, por exemplo, conforme dados em um banco de dados.

O Apache percebe quando um servidor é estático ou dinâmico, dessa forma, ele sabe quando deve buscar por dados ou não.

### Por que usar o Apache?
O Apache domina mais de 50% de todo o mercado, mesmo tendo sido superado pelo Nginx, além de ter uma curva de aprendizado relativamente simples em relação à outros servidores.

## Como instalar o Apache e o DNS em um contêiner
Utilizaremos o Docker para instalar a imagem dentro de um contêiner virtual.

Primeiro, crie um diretório dentro de alguma pasta do seu computador com o nome `apache-server`

Então, abra o CMD na pasta e digite o comando:

```bash
docker run -d --name srv-apache-bind -p 8080:80 httpd
```

Após isso, entre na imagem com o comando:

```bash
docker exec -it srv-apache-bind /bin/bash
```

Agora dentro da imagem, vamos começar instalando os programas necessários além do Apache.

```bash
apt-get update && apt-get install bind9 bind9utils dnsutils vim -y
```

Depois disso, tudo que utilizaremos será instalado.

### Configurando Apache

Vamos começar indo até o arquivo 

```bash
vim /usr/local/apache2/conf/httpd.conf
```

Dentro desse arquivo, vamos configurar as pastas dos 2 sites que vamos utilizar.

```bash
<VirtualHost *:80>
    ServerName wegone.site.lan
    ServerAlias wegone.localhost
    DocumentRoot "/usr/local/apache2/htdocs/wegone"
    
    <Directory "/usr/local/apache2/htdocs/wegone">
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>

<VirtualHost *:80>
    ServerName gundes.site.lan
    ServerAlias gundes.localhost
    DocumentRoot "/usr/local/apache2/htdocs/gundes"

    DirectoryIndex index.php index.html

    <Directory "/usr/local/apache2/htdocs/gundes">
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

Também altere a linha 52 do arquivo, deve ser `Listen 80`

Por padrão, o Listen virá na porta `8080`, mas caso essa porta esteja bloqueada, troque para `8081`.

Após isso, salve e vá até o diretório

```bash
cd /usr/local/apache2/htdocs
```

Crie duas pastas, uma chamada `wegone` e outra chamada `gundes`

Dentro destes diretórios, você precisa criar o `index.html`, crie seu site lá

> Você inclusive pode criar pastas e sites diferentes, apenas altere no `DocumentRoot` e `<Directory>`, dentro do arquivo `httpd.conf`.

Com isso feito, a configuração do apache está completa.

### Configurando BIND

Vamos até o diretório do Bind

```bash
cd /etc/bind
```

Aqui, vamos editar os arquivos `named.conf.local` e `named.conf.options`.

```bash
// named.conf.local

zone "site.lan" {
    type master;

    file "/etc/bind/zones/db.site.lan";
};

zone "0.168.192.in-addr.arpa" {
    type master;

    file "/etc/bind/zones/db.192.168";
};
```

```bash
// named.conf.options

options {
    directory "/var/cache/bind";

    forwarders {
        8.8.8.8;
    };

    listen-on { any; };

    allow-query { any; };

    dnssec-validation auto;

    listen-on-v6 { any; };

    recursion yes;
};
```

Após isso, crie um novo diretório dentro da pasta do Bind

```bash
mkdir zones && cd zones
```

Dentro dessa pasta, crie dois arquivos: `db.site.lan` e `db.192.168`

> Você pode alterar o nome e o IP, mas lembre-se de fazer o mesmo no `named.conf.local`

Agora, vamos editar estes arquivos.

```bash
; db.site.lan
$TTL    604800
@       IN      SOA     ns1.site.lan. admin.site.lan. (
                        2026022401
                        86400
                        64800
                        2419200
                        604800 )

@       IN      NS      ns1.site.lan.
ns1     IN      A       192.168.0.1
wegone  IN      A       192.168.0.100
gundes  IN      A       192.168.0.200
```

```bash
; db.192.168
$TTL    604800
@       IN      SOA     ns1.site.lan. admin.site.lan. (
                        2026022401
                        86400
                        64800
                        2419200
                        604800 )

@       IN      NS      ns1.site.lan.
100     IN      PTR     wegone.site.lan.
200     IN      PTR     gundes.site.lan.
```

Após isso, as configurações do Bind estão feitas.

Agora, comece o serviço do Bind, então teste usando o comando `dig`

```bash
service named start

dig @localhost -x 192.168.0.100
dig @localhost wegone.site.lan
```

Os comandos devem retornar com status `[ OK ]` e `NOERROR`

Para finalizar, abra seu navegador e entre em algum virtual host, ou seja, `gundes.localhost:8080` ou `wegone.localhost:8080`

> O `site.lan` **não** funcionará fora do Docker, pois as configurações do PC não perguntam diretamente ao DNS do seu contêiner no Docker.
