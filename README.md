# PostgreSQL as Vector Database

How to create a vector store from scratch

The instructions are based on a Linux server

# Table of Contents

- Prerequisites
  - [Install Docker](#install-docker-engine)
  - [Install PostgreSQL & pgvector](#install-postgresql-pgvector)
  - [Firewall Configuration](#firewall-configuration)
  - [Setup pgAdmin](#setup-pgadmin)
  - [Load data into DB](#load-data-into-db)
  - [Retrieve data from DB](#retrieve-data-from-db)

---

## [install-docker-engine](https://docs.docker.com/engine/install/)

#### Set up Docker's apt repository.

```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

#### Install the Docker packages.

```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

---

## install-PostgreSQL-pgvector

A good thing about this image is that it comes with pgvector preinstalled which saves tons of time.

#### Create a file named _docker-compose.yml_

```
version: "3.8"
services:
  db:
    image: ankane/pgvector:latest
    container_name: local_pgdb
    restart: always
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: POSTGRES_USER
      POSTGRES_PASSWORD: POSTGRES_PASSWORD
    volumes:
      - local_pgdata:/var/lib/postgresql/data
  pgadmin:
    image: dpage/pgadmin4
    container_name: pgadmin4_container
    restart: always
    ports:
      - "8888:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: PGADMIN_DEFAULT_EMAIL
      PGADMIN_DEFAULT_PASSWORD: PGADMIN_DEFAULT_PASSWORD
    volumes:
      - pgadmin-data:/var/lib/pgadmin

volumes:
  local_pgdata:
  pgadmin-data:
```

#### Installation part

```
docker compose up -d
```

#### Install pgvector extension

```
docker exec -it local_pgdb psql -U labadmin -c 'CREATE EXTENSION vector'
```

Now you have successfully installed both

## firewall-configuration

#### Based on your Cloud Computing Provider, make sure to configure security groups to allow incoming connections to PostgreSQL and pgAdmin.

#### If you’re using AWS Security Groups, make sure the security group associated with your EC2 instance allows incoming connections on port 5432 TCP (the default PostgreSQL port) and 8888 TCP (pgAdmin port)

---

## setup-pgadmin

#### Now you can access pgAdmin on your server's public IP on pgAdmin port which is 8888 (*SERVER_PUBLIC_IP:8888*). You can log in with the *PGADMIN_DEFAULT_EMAIL* and *PGADMIN_PASSWORD* you set in *docker-compose.yml* earlier.

#### Right-click on Servers and register a new server. Give it a name and in the Connection tab, add your hostname and set your username and password based on the settings you set in your *docker-compose.yml* file. (*POSTGRES_USER, POSTGRES_PASSWORD*)

<img src="images/pgAdmin.png" width="600" height="500" />

---

## [load-data-into-db](https://python.langchain.com/docs/integrations/vectorstores/pgvector/)

#### I import data into DB using LangChain’s PGVector module which makes it so easy to import and retrieve data.

#### Install dependencies

```
!pip install --quiet -U langchain_cohere
!pip install --quiet -U langchain_postgres
```

#### Initialize the vectorstore

You can use any embedding (HuggingFace) model you like, check out LangChain's documentation for that.

```
from langchain_cohere import CohereEmbeddings
from langchain_core.documents import Document
from langchain_postgres import PGVector
from langchain_postgres.vectorstores import PGVector

# See docker command above to launch a postgres instance with pgvector enabled.
connection = "postgresql+psycopg://langchain:langchain@localhost:6024/langchain"  # Uses psycopg3!
collection_name = "my_docs"
embeddings = CohereEmbeddings()

vectorstore = PGVector(
    embeddings=embeddings,
    collection_name=collection_name,
    connection=connection,
    use_jsonb=True,
)
```

#### Add documents

```
docs = [
    Document(
        page_content="there are cats in the pond",
        metadata={"id": 1, "location": "pond", "topic": "animals"},
    ),
    Document(
        page_content="ducks are also found in the pond",
        metadata={"id": 2, "location": "pond", "topic": "animals"},
    ),
    Document(
        page_content="fresh apples are available at the market",
        metadata={"id": 3, "location": "market", "topic": "food"},
    )
]
```

```
vectorstore.add_documents(docs, ids=[doc.metadata["id"] for doc in docs])
```

---

## [retrieve-data-from-db](https://python.langchain.com/docs/integrations/vectorstores/pgvector/)

```
vectorstore.similarity_search("cat", k=`)
>>> [Document(page_content='there are cats in the pond', metadata={'id': 1, 'topic': 'animals', 'location': 'pond'})]
```
