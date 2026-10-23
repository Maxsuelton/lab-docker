# lab-docker

Laboratório de Docker em três etapas: construção de imagem própria, persistência de dados com volume nomeado e comunicação entre containers pela rede do Compose.

Todos os testes rodam localmente em WSL2 + Docker Desktop, sem dependência de nuvem ou registry externo.

## Pré-requisitos

- Docker Desktop com integração WSL2 habilitada
- Docker Compose v2 (comando `docker compose`, sem hífen)

## Estrutura do repositório

```text
lab-docker/
├── etapa1/
│   ├── Dockerfile
│   └── index.html
├── etapa2/
│   └── docker-compose.yml
├── etapa3/
│   └── docker-compose.yml
└── prints/
```

## Etapa 1 — Imagem própria

**Objetivo:** construir uma imagem a partir de um Dockerfile, em vez de consumir imagem pronta do registry.

```bash
cd etapa1
docker build -t meu-nginx:1.0 .
docker run -d --name web -p 8080:80 meu-nginx:1.0
docker ps
```

**Resultado:** container `web` em execução com a imagem `meu-nginx:1.0`, status `Up` e mapeamento `0.0.0.0:8080->80/tcp`. A página é servida em `localhost:8080`, a partir do `index.html` copiado pelo `COPY` no build.

![Container em execução](prints/etapa1-docker-ps.png)

![Página servida em localhost:8080](prints/etapa1-navegador.png)

**Conclusão:** imagem é o artefato imutável; container é a instância em execução. O `-p 8080:80` faz o mapeamento entre a porta do host e a do container — sem ele, o serviço existe mas não é alcançável.

## Etapa 2 — Persistência

**Objetivo:** entender o ciclo de vida dos dados em container e o papel do volume nomeado.

```bash
cd etapa2
docker compose up -d
docker exec -it lab-postgres psql -U postgres -d lab -c "CREATE TABLE clientes (id serial, nome text);"
docker exec -it lab-postgres psql -U postgres -d lab -c "INSERT INTO clientes (nome) VALUES ('Maria');"
docker exec -it lab-postgres psql -U postgres -d lab -c "SELECT * FROM clientes;"
```

**Resultado:** tabela `clientes` criada e registro `Maria` inserido. O `SELECT` retorna `1 | Maria`.

### Teste 1 — parar sem apagar o volume

```bash
docker compose down
docker compose up -d
docker exec -it lab-postgres psql -U postgres -d lab -c "SELECT * FROM clientes;"
```

**Resultado:** o registro permaneceu. O `down` removeu o container, mas o volume nomeado sobreviveu.

![Dados persistidos após down](prints/etapa2-dados-persistidos.png)

### Teste 2 — apagar o volume

```bash
docker compose down -v
docker compose up -d
docker exec -it lab-postgres psql -U postgres -d lab -c "SELECT * FROM clientes;"
```

**Resultado:** `ERROR: relation "clientes" does not exist`. O `down -v` destruiu o volume, e o banco subiu vazio.

![Erro após down -v](prints/etapa2-down-v-erro.png)

**Conclusão:** `down` para o serviço e preserva os dados. `down -v` destrói o volume junto. Em ambiente real, essa diferença é a linha entre reiniciar um serviço e perder um banco.

## Etapa 3 — Rede entre containers

**Objetivo:** fazer dois containers se comunicarem pela rede interna do Compose, sem expor o banco ao host.

```bash
cd etapa3
docker compose up -d
docker compose ps
docker exec -it lab-postgres pg_isready -U postgres -d lab
```

**Resultado:** rede `lab_lab_net` criada, `lab-postgres` em `Up (healthy)` e `lab-adminer` em `Up` na porta `8081`. O `pg_isready` retornou `accepting connections`.

![Containers no ar com Postgres healthy](prints/etapa3-compose-up.png)

Acesso ao Adminer em `localhost:8081` com sistema `PostgreSQL`, servidor `db`, usuário `postgres`, senha `senha123` e base `lab`. A base estava vazia — o volume havia sido removido no `down -v` da etapa anterior.

<img width="1915" height="676" alt="etapa3-adminer" src="https://github.com/user-attachments/assets/adcf117c-676f-423e-850c-1d075312e44a" />

**Conclusão:** dentro da rede do Compose, cada serviço é resolvido pelo próprio nome — por isso o campo servidor recebe `db`, e não `localhost`. O Postgres não expõe a porta 5432 ao host porque apenas o Adminer se conecta a ele, pela rede interna. O `depends_on` com `condition: service_healthy` garantiu que o Adminer só subisse depois que o banco respondesse ao `pg_isready`.

## Resumo do que foi praticado

| Etapa | Conceito | Evidência |
|---|---|---|
| 1 | Build de imagem própria com Dockerfile | `docker ps` e página em `localhost:8080` |
| 2 | Persistência com volume nomeado | Dados sobrevivem ao `down`, somem no `down -v` |
| 3 | Rede interna e resolução por nome de serviço | Adminer conectando em `db`, porta 5432 não exposta |

## Autor

Maxsuelton Pinheiro — laboratório de estudos em infraestrutura e containers.



