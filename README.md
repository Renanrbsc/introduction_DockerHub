Introdução ao Docker Compose

Tempo estimado de leitura: 10 minutos

Nesta página, você cria um aplicativo Web Python simples em execução no Docker Compose. O aplicativo usa a estrutura do Flask e mantém um contador de visitas no Redis.

Pré-requisitos 
Verifique se você já instalou o Docker Engine e o Docker Compose . Você não precisa instalar o Python ou o Redis, pois ambos são fornecidos pelas imagens do Docker.

Etapa 1: Configuração 
Defina as dependências do aplicativo.

Crie um diretório para o projeto:

$ cd introduction_dockerhub
Crie um arquivo chamado app.py no diretório do projeto e cole o seguinte script:

'''
    import time

    import redis
    from flask import Flask

    app = Flask(__name__)
    cache = redis.Redis(host='redis', port=6379)


    def get_hit_count():
        retries = 5
        while True:
            try:
                return cache.incr('hits')
            except redis.exceptions.ConnectionError as exc:
                if retries == 0:
                    raise exc
                retries -= 1
                time.sleep(0.5)


    @app.route('/')
    def hello():
        count = get_hit_count()
        return 'Hello World! I have been seen {} times.\n'.format(count)
'''

Neste exemplo, redis é o nome do host do contêiner redis na rede do aplicativo. Usamos a porta padrão para Redis 6379.

"
    Tratamento de erros transitórios

    Observe como a get_hit_countfunção é escrita. Esse loop de repetição básico nos permite tentar nossa solicitação várias vezes se o serviço redis não estiver disponível. Isso é útil na inicialização enquanto o aplicativo fica on-line, mas também o torna mais resistente se o serviço Redis precisar ser reiniciado a qualquer momento durante a vida útil do aplicativo. Em um cluster, isso também ajuda a lidar com quedas momentâneas de conexão entre nós.
"

'''
    Crie outro arquivo chamado requirements.txt no diretório do projeto e cole o seguinte script:

    flask
    redis
'''


Passo 2: Criar um Dockerfile 

Nesta etapa, você escreve um Dockerfile que cria uma imagem do Docker. A imagem contém todas as dependências exigidas pelo aplicativo Python, incluindo o próprio Python.

No diretório do projeto, crie um arquivo chamado Dockerfilee cole o seguinte script:
'''
    FROM python:3.7-alpine
    WORKDIR /code
    ENV FLASK_APP app.py
    ENV FLASK_RUN_HOST 0.0.0.0
    RUN apk add --no-cache gcc musl-dev linux-headers
    COPY requirements.txt requirements.txt
    RUN pip install -r requirements.txt
    COPY . .
    CMD ["flask", "run"]
'''
Isso diz ao Docker para:

Crie uma imagem começando com a imagem Python 3.7.
Defina o diretório de trabalho como /code.
Defina variáveis ​​de ambiente usadas pelo flaskcomando.
Instale o gcc para que pacotes Python, como MarkupSafe e SQLAlchemy, possam compilar acelerações.
Copie requirements.txte instale as dependências do Python.
Copie o diretório atual .no projeto para a área de trabalho .na imagem.
Defina o comando padrão para o contêiner flask run.


Passo 3: Definir os serviços em um arquivo Compose 

Crie um arquivo chamado docker-compose.yml no diretório do projeto e cole o seguinte script:
'''
    version: '3'
    services:
    web:
        build: .
        ports:
        - "5000:5000"
    redis:
        image: "redis:alpine"
'''
Este arquivo de composição define dois serviços: web e redis.

Serviço Web 
O webserviço usa uma imagem criada a partir do Dockerfile diretório atual. Em seguida, vincula o contêiner e a máquina host à porta exposta 5000,. Este serviço de exemplo usa a porta padrão para o servidor da web Flask 5000.

Serviço Redis 
O serviço redis usa uma imagem pública do Redis extraída do registro do Docker Hub.


Passo 4: Criar e executar seu aplicativo com o Compose 
No diretório do projeto, inicie o aplicativo executando docker-compose up.

A composição puxa uma imagem Redis, cria uma imagem para o seu código e inicia os serviços que você definiu. Nesse caso, o código é copiado estaticamente na imagem no momento da criação.

Digite http://localhost:5000/ em um navegador para ver o aplicativo em execução.

Você deve ver uma mensagem no seu navegador dizendo:

Hello World! I have been seen 1 times.
olá mundo no navegador

Recarregue a página.

O número deve aumentar.

Hello World! I have been seen 2 times.
olá mundo no navegador

Alterne para outra janela do terminal e digite docker image ls para listar imagens locais.

Listar imagens neste momento deve retornar redise web.

$ docker image ls
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
composetest_web         latest              e2c21aa48cc1        4 minutes ago       93.8MB
python                  3.4-alpine          84e6077c7ab6        7 days ago          82.5MB
redis                   alpine              9d8fa9aa0e5b        3 weeks ago         27.5MB
Você pode inspecionar imagens com docker inspect <tag or id>.

Interrompa o aplicativo executando docker-compose down no diretório do projeto no segundo terminal ou pressionando CTRL + C no terminal original em que você iniciou o aplicativo.

Passo 5: Edite o arquivo Compose para adicionar um ligamento 
Edite docker-compose.yml no diretório do projeto para adicionar uma montagem de ligação para o webserviço:
'''
    version: '3'
    services:
    web:
        build: .
        ports:
        - "5000:5000"
        volumes:
        - .:/code
        environment:
        FLASK_ENV: development
    redis:
        image: "redis:alpine"
'''

A nova chave volumes monta o diretório do projeto (diretório atual) no host para /code dentro do contêiner, permitindo modificar o código rapidamente, sem a necessidade de reconstruir a imagem. A environment chave define a variável de ambiente FLASK_ENV, que informa flask run para executar no modo de desenvolvimento e recarregar o código na alteração. Este modo deve ser usado apenas no desenvolvimento.

Passo 6: Re-criar e executar o aplicativo com o Compose 
No diretório do projeto, digite docker-compose up para criar o aplicativo com o arquivo de composição atualizado e execute-o.

$ docker-compose up
Creating network "composetest_default" with the default driver
Creating composetest_web_1 ...
Creating composetest_redis_1 ...
Creating composetest_web_1
Creating composetest_redis_1 ... done
Attaching to composetest_web_1, composetest_redis_1
web_1    |  * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
...
Verifique a Hello Worldmensagem em um navegador da Web novamente e atualize para ver o incremento da contagem.

Pastas compartilhadas, volumes e montagens de ligação


Passo 7: Atualize o aplicativo 
Como o código do aplicativo agora está montado no contêiner usando um volume, você pode fazer alterações no código e ver as alterações instantaneamente, sem precisar reconstruir a imagem.

Mude a saudação app.py e salve-a. Por exemplo, altere a Hello World! mensagem para Hello from Docker!:

return 'Hello from Docker! I have been seen {} times.\n'.format(count)
Atualize o aplicativo no seu navegador. A saudação deve ser atualizada e o contador ainda deve estar aumentando.

olá mundo no navegador

Passo 8: Experiência com alguns outros comandos 
Se você deseja executar seus serviços em segundo plano, pode passar a tag -d  (para o modo "desanexado") docker-compose up e usar docker-compose ps para ver o que está em execução no momento:

$ docker-compose up -d
Starting composetest_redis_1...
Starting composetest_web_1...

$ docker-compose ps
Name                 Command            State       Ports
-------------------------------------------------------------------
composetest_redis_1   /usr/local/bin/run         Up
composetest_web_1     /bin/sh -c python app.py   Up      5000->5000/tcp

O docker-compose run comando permite executar comandos únicos para seus serviços. Por exemplo, para ver quais variáveis ​​de ambiente estão disponíveis para o webserviço:

$ docker-compose run web env
Veja docker-compose --help para ver outros comandos disponíveis. Você também pode instalar a conclusão do comando para o shell bash e zsh, que também mostra os comandos disponíveis.

Se você iniciou o Compose docker-compose up -d, interrompa seus serviços quando terminar com eles:

$ docker-compose stop
Você pode derrubar tudo, removendo os contêineres inteiramente, com o down comando Passe --volumes também para remover o volume de dados usado pelo contêiner Redis:

$ docker-compose down --volumes
Neste ponto, você já viu o básico de como o Compose funciona.