# docker-wsl

Configurar o Docker no WSL

Certifique-se de que o Docker Desktop está integrado ao WSL:
- Abra o Docker Desktop no Windows.
- Vá até `Settings > Resources > WSL Integration`.
- Habilite a distribuição Linux que você está usando (exemplo: Ubuntu).

Use o Docker diretamente no WSL:
        
No terminal do WSL, verifique se o Docker está funcionando:
```
docker --version
```
Teste se o Docker está rodando corretamente:
```
docker run hello-world
```
Se os códigos do Frontend e do Backend estão no windows, é preciso levar para a pasta do Linux no WSL:

Crie um diretório para o projeto no WSL:
```
mkdir -p ~/projects/diretorio_projeto
```
No terminal do WSL, você pode usar o comando `cp` para copiar os arquivos do Windows para o Linux no WSL. O caminho do Windows deve ser convertido para o formato do WSL.

Dado o seu caminho do Windows:
```
C:\Users\pedro\Documents\SAAS\novo codigo\Docker\diretorio_projeto
```
No terminal do WSL, use o caminho do sistema de arquivos do Windows para copiar os arquivos:
```
cp -r /mnt/c/Users/pedro/Documents/SAAS/novo\ codigo/Docker/diretorio_projeto/* ~/projects/diretorio_projeto/
```
Após a cópia, confira os arquivos na pasta de destino:
```
ls ~/projects/diretorio_projeto
```

**Explicação**

`/mnt/c:` O sistema de arquivos do Windows é montado no WSL sob `/mnt/c`. <br>
`Espaço no caminho:` A barra invertida `(\)` foi usada porque o nome da pasta "novo codigo" contém um espaço, e no terminal Linux (ou no WSL), 
é necessário escapar os espaços em nomes de arquivos ou pastas para que o comando seja interpretado corretamente.<br>
`~/projects/diretorio_projeto/:` Diretório de destino no WSL onde você criou seu projeto. <br>

## Backend

Para o Backend do exemplo, é necessário ter instalado o Java 17 e o Maven.<br>

Certifique-se de que seu projeto já foi compilado e que o JAR está no diretório `target/`. No diretório raiz onde está o POM, compile e crie o `.jar`, execute o comando:
```
mvn clean package
```
Agora, para teste, rode o Backend usando o seguinte comando:
```
java -jar target/nomedoprojeto-0.0.1-SNAPSHOT.jar
```

**1. Dockerizar o Backend**
 
*1.1 Criar o Dockerfile para o Backend*

No diretório do Backend (onde está o `pom.xml`), crie um arquivo chamado Dockerfile com o seguinte conteúdo:

```
# Usar uma imagem base do OpenJDK 17
FROM openjdk:17-jdk-slim

# Definir o diretório de trabalho dentro do container
WORKDIR /app

# Copiar o arquivo JAR gerado para o container
COPY target/nomedoprojeto-0.0.1-SNAPSHOT.jar app.jar

# Expor a porta 8080 do container
EXPOSE 8080

# Comando para rodar a aplicação no container
CMD ["java", "-jar", "app.jar"]

```

*1.2 Construir a Imagem Docker do Backend*

No terminal, para teste, no diretório do Backend (onde está o Dockerfile), construa a imagem Docker:
```
docker build -t diretorio_backend .
```

## Frontend

*Passo 1: Acesse o Diretório do Frontend*

Entre na pasta do Frontend no WSL:
```
cd ~/projects/diretorio_projeto/diretorio_frontend
```
*Passo 2: Instale as Dependências*

Se ainda não instalou as dependências, rode:
```
npm install
```
Isso instalará todas as bibliotecas necessárias descritas no `package.json`.

*Ajustando o `package.json`*

Aqui está um exemplo de como o seu `package.json` pode ser ajustado na parte de scripts, para funcionar com o Docker:
```
{
  "name": "diretorio_frontend",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "start": "node server.js",       // Comando para iniciar o servidor
    "build": "echo 'No build step'"  // Apenas um placeholder, já que você não tem processo de build
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "description": "",
  "dependencies": {
    "express": "^4.21.1"
  }
}
```
*Passo 3: Inicie o Servidor do Frontend*

Para rodar o Frontend, a modo de teste, use o comando:
```
npm start
```

**2.1 Criar o Dockerfile para o Frontend**

No diretório do Frontend (onde está o `package.json`), crie um arquivo chamado Dockerfile com o seguinte conteúdo:

```
# Usar uma imagem base do Node.js
FROM node:18-alpine

# Definir o diretório de trabalho dentro do container
WORKDIR /app

# Copiar os arquivos do frontend para o container
COPY package*.json ./
COPY . .

# Instalar as dependências do frontend
RUN npm install

# Expor a porta 4200 do container
EXPOSE 4200

# Comando para rodar o Nginx
CMD ["npm", "start"]

```
**2.2 Construir a Imagem Docker do Frontend**

No terminal, a modo de teste, no diretório do Frontend (onde está o Dockerfile), construa a imagem Docker:
```
docker build -t diretorio_frontend .
```

**3. Orquestrando a Dockerização**

No diretório raiz do projeto (ou onde desejar), crie um arquivo chamado `docker-compose.yml` para orquestrar a Dockerização com o seguinte conteúdo:
```
version: '3.8'

services:
  database:
    image: mysql:8.0
    container_name: mysql_database
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: 123456
      MYSQL_DATABASE: main_database
    volumes:
      - /home/pedro/db_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 3

  backend:
    build:
      context: ./diretorio_backend
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://database:3306/main_database
      SPRING_DATASOURCE_USERNAME: root
      SPRING_DATASOURCE_PASSWORD: 123456
      SPRING_DATASOURCE_DRIVER_CLASS_NAME: com.mysql.cj.jdbc.Driver
      SPRING_JPA_HIBERNATE_DDL_AUTO: update
      SPRING_JPA_SHOW_SQL: "true"
      SPRING_JPA_PROPERTIES_HIBERNATE_DIALECT: org.hibernate.dialect.MySQL8Dialect
      SPRING_DATASOURCE_HIKARI_CONNECTION_TIMEOUT: 30000
      SPRING_DATASOURCE_HIKARI_MINIMUM_IDLE: 5
      SPRING_DATASOURCE_HIKARI_MAXIMUM_POOL_SIZE: 10
    depends_on:
      - database

  frontend:
    build:
      context: ./diretorio_frontend
    ports:
      - "4200:4200"

volumes:
  db_data:
```

**4. Executar os Containers**

No terminal, navegue até o diretório onde está o `docker-compose.yml`.

Execute o seguinte comando para iniciar todos os serviços:
```
docker-compose up --build
```
Você verá os logs dos três serviços: Backend, Frontend e banco de dados.

Dockerizar seu projeto significa criar imagens Docker para o Frontend e o Backend. Essas imagens são "blueprints" que podem ser executadas como containers, encapsulando tudo o que o Backend e o Frontend precisam para rodar, como configurações de ambiente e dependências.

Quando você tiver as imagens prontas, poderá usá-las para rodar o Frontend e o Backend em containers separados, isolados do sistema host, mas que funcionam juntos.

**5. Testar os Containers**

Acesse o Frontend no navegador:
```
http://localhost:4200
```
O Backend deve estar rodando na porta 8080:
```
http://localhost:8080
```
Verifique se o Frontend está se comunicando corretamente com o Backend (ex.: realizar operações no Frontend e monitorar os logs do Backend).

**6. Parar os Containers**

Quando quiser parar os containers, use:
```
docker-compose down
``` 
