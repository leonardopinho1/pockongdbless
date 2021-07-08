## kong

# Instalação do Kong no Docker Compose em modo declarativo


### O que é Kong?

Você pode encontrar a distribuição oficial do Docker para Kong em [https://hub.docker.com/_/kong](https://hub.docker.com/_/kong).

### Instalando o kong

1 - Execute o arquivo dockercompose.yaml em sua IDE de prefenrencia ou até mesmo no seu terminal utilizando o comando **docker-compose up -d **. Para validar se o container esta up e funcional utilizando o comando **docker ps** ou **docker-compose ps**. 

O resultado deve ser igual ou semelhante a imagem abaixo, o objetivo desse comando é validar se os containers estão up e funcionais.

![Captura de tela 2021-03-24 222823](https://user-images.githubusercontent.com/68164552/112405075-591c5200-8cf0-11eb-9914-aaecfb96a0a2.jpg)

Feito todos esses passo vou explicar detalhes do manifesto informando o que foi feito nessa POC.

### Explicando o Manifesto

Nessa POC subimos 2 aplicações KONG (kong1 e kong 2) para validarmos a redundância entre os kongs conforme código abaixo.

```services:    
  kong1:
    image: kong:latest
    volumes:
      - ./declarative:/usr/local/kong/declarative
    environment:    
      - KONG_DATABASE=off
      - KONG_DECLARATIVE_CONFIG=/usr/local/kong/declarative/kong.yaml
      - KONG_PROXY_ACCESS_LOG=/dev/stdout
      - KONG_ADMIN_ACCESS_LOG=/dev/stdout
      - KONG_PROXY_ERROR_LOG=/dev/stderr
      - KONG_ADMIN_ERROR_LOG=/dev/stderr
      - KONG_ADMIN_LISTEN=0.0.0.0:8001
  
    mem_limit: "3g"
    memswap_limit: "3g"
    cpus: 4
    logging:
      options:
        max-size: "50m"
    networks:
      - kong-net
    restart: always

kong2:
    image: kong:latest
    volumes:
      - ./declarative:/usr/local/kong/declarative
    environment:    
      - KONG_DATABASE=off
      - KONG_DECLARATIVE_CONFIG=/usr/local/kong/declarative/kong.yaml
      - KONG_PROXY_ACCESS_LOG=/dev/stdout
      - KONG_ADMIN_ACCESS_LOG=/dev/stdout
      - KONG_PROXY_ERROR_LOG=/dev/stderr
      - KONG_ADMIN_ERROR_LOG=/dev/stderr
      - KONG_ADMIN_LISTEN=0.0.0.0:8001
  
    mem_limit: "3g"
    memswap_limit: "3g"
    cpus: 4
    logging:
      options:
        max-size: "50m"
    networks:
      - kong-net
    restart: always```

Observe que na linha 27 especificamos o volume que no caso é o nosso arquivo declarativo, como não utilizamos banco de dados no modo dbless é nesse volume que vamos especificar o código yaml que esta configurado nossa API com o Basic Auth.
  
 ```_format_version: "2.1"
_transform: true

services:
- connect_timeout: 60000
  host: mocktarget.apigee.net
  name: mock
  port: 443
  protocol: https
  read_timeout: 60000
  retries: 3  
  write_timeout: 60000
  routes:
  - name: routemock
    paths:
    - /mock
    path_handling: v1
    preserve_host: true
    protocols:
    - http
    - https
    regex_priority: 0
    strip_path: true
    https_redirect_status_code: 426
  plugins:
  - name: basic-auth
    route:
    config: 
      hide_credentials: true
consumers:
- username: leo_arch
  custom_id: SOME_CUSTOM_ID 

basicauth_credentials:
- consumer: leo_arch
  username: leo
  password: leo_123_asdf*```

E para fazer esse balanceamento decarga utilizamos o NGINX como servidor web em modo load balance e utilizamos o least connect como estratégia de balanceamento de carga, que tem como objetivo direcionar conexões para o servidor que tiver com a menor ocupação.

 ```nginx:
    image: nginx
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./logs:/nginx-logs
    ports:
      - "8080:8080"
   
    mem_limit: "2g"
    memswap_limit: "2g"
    cpus: 4
    logging:
      options:
        max-size: "50m"
    networks:
      - kong-net
    restart: always```
    
E claro, para esse load balance funcionar precisamos utilizar o arquivo de configuração nginx.conf

```worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;
events {
    worker_connections 768;
}
http {
    ##
    # Basic Settings
    ##
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 128;
    types_hash_max_size 2048;
    server_tokens off;
    # server_names_hash_bucket_size 64;
    # server_name_in_redirect off;
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    ##
    # SSL Settings
    ##
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
    ssl_prefer_server_ciphers on;
    ##
    # Logging Settings
    ##
    # access_log /var/log/nginx/access.log;
    # error_log /var/log/nginx/error.log;
    ##
    # Gzip Settings
    ##
    gzip on;
    # gzip_vary on;
    # gzip_proxied any;
    # gzip_comp_level 6;
    # gzip_buffers 16 8k;
    # gzip_http_version 1.1;
    # gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
    ##
    # Virtual Host Configs
    ##
    upstream kong_upstream {
        least_conn;
        server kong1:8000;
        server kong2:8000;
    }
    server { # simple load balancing
        listen          8080;
        #server_name     big.server.com;
        #access_log      logs/big.server.access.log main;

        location / {
            proxy_pass  http://kong_upstream;
        }
    }
                

}```

Veja que no arquivo de configuração especificamos a porta 8000 (padrão do kong) em 2 servidores (kong1 e kong2) utilizando o load balance least connect.

Para validarmos a funcionalidade da nossa API vamos digitar no navegador o endereço do nginx + o path da nossa API localhost:8080/mock

![Captura de tela 2021-03-24 225759](https://user-images.githubusercontent.com/68164552/112407194-6f2c1180-8cf4-11eb-9080-0e515078ff42.jpg)

Veja que conforme o esperado a API solicitou usuário e senha, pois estamos utilizando o Basic auth como meio de autenticação, poderiamos inserir o usuário e senha das linhas 108 e 109 no navegador mesmo que iria funcionar tranquilamente, porém o ideal é realizar esse teste no postman.

![Captura de tela 2021-03-24 230417](https://user-images.githubusercontent.com/68164552/112407691-507a4a80-8cf5-11eb-9a00-84ed6cd19a4b.jpg)

Veja que passamos as credenciais e deu tudo certo.

Pronto! Finalizamos essa POC com todos os objetivos atingidos.

A documentação de Kong pode ser encontrada em [https://docs.konghq.com/][kong-docs-url].

##DOCS adicionais

[kong-site-url]: https://konghq.com/
[kong-docs-url]: https://docs.konghq.com
