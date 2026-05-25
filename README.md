# PERSISTENCIA DE DADOS E GERENCIAMENTO DE VOLUMES EM CONTAINERS 
Passo Inicial — Preparação do Ambiente
Validar Instalações
docker --version
docker compose version
git --version

Testar Docker
docker run hello-world

Cenário 1 — Persistência de Dados com MySQL e Named Volume
1. Criar Volume Docker
docker volume create mysql-prod-data
Verificar:
docker volume ls

2. Criar Container MySQL
docker run -d \
--name mysql-prod \
-e MYSQL_ROOT_PASSWORD=123456 \
-e MYSQL_DATABASE=empresa \
-v mysql-prod-data:/var/lib/mysql \
-p 3306:3306 \
mysql:8.0
Verificar container:
docker ps

3. Acessar MySQL
docker exec -it mysql-prod mysql -u root -p
Senha:
123456

4. Criar Tabela e Inserir Dados
USE empresa;

CREATE TABLE usuarios (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nome VARCHAR(100),
    email VARCHAR(100)
);

INSERT INTO usuarios (nome, email)
VALUES
('Joao', 'joao@email.com'),
('Maria', 'maria@email.com'),
('Carlos', 'carlos@email.com');

SELECT * FROM usuarios;

5. Validar Persistência
Remover Container
docker rm -f mysql-prod
Recriar Container Utilizando Mesmo Volume
docker run -d \
--name mysql-prod \
-e MYSQL_ROOT_PASSWORD=123456 \
-v mysql-prod-data:/var/lib/mysql \
-p 3306:3306 \
mysql:8.0
Validar Dados
docker exec -it mysql-prod mysql -u root -p
USE empresa;
SELECT * FROM usuarios;

Explicação Técnica — Cenário 1
·Containers são efêmeros. 
·O volume mysql-prod-data armazena os dados fora do ciclo de vida do container. 
·Mesmo removendo o container, os dados permanecem no volume Docker. 

Cenário 2 — Backup e Restauração de Volume
1. Criar Pasta de Backup
mkdir backups

2. Backup com mysqldump
docker exec mysql-prod /usr/bin/mysqldump -u root --password=123456 empresa > backups/empresa.sql

3. Backup do Volume
docker run --rm \
-v mysql-prod-data:/volume \
-v $(pwd)/backups:/backup \
ubuntu \
tar czvf /backup/mysql-volume.tar.gz /volume

4. Simular Perda do Volume
docker rm -f mysql-prod
docker volume rm mysql-prod-data

5. Criar Novo Volume
docker volume create mysql-prod-data

6. Restaurar Backup do Volume
docker run --rm \
-v mysql-prod-data:/volume \
-v $(pwd)/backups:/backup \
ubuntu \
bash -c "cd /volume && tar xzvf /backup/mysql-volume.tar.gz --strip 1"

7. Subir Novo Container
docker run -d \
--name mysql-prod \
-e MYSQL_ROOT_PASSWORD=123456 \
-v mysql-prod-data:/var/lib/mysql \
-p 3306:3306 \
mysql:8.0

8. Validar Dados
docker exec -it mysql-prod mysql -u root -p
USE empresa;
SELECT * FROM usuarios;

Explicação Técnica — Cenário 2
·mysqldump realiza backup lógico do banco. 
·tar.gz realiza backup físico do volume. 
·O processo demonstra recuperação de desastre. 

Cenário 3 — Bind Mount
1. Criar Diretório Local
mkdir app-dev

2. Criar Arquivo Local
echo "Arquivo criado no host" > app-dev/teste.txt

3. Executar Container com Bind Mount
docker run -it --name ubuntu-bind \
-v $(pwd)/app-dev:/dados \
ubuntu bash

4. Validar Dentro do Container
cd /dados
ls
cat teste.txt

5. Criar Arquivo no Container
echo "Arquivo criado no container" > container.txt

6. Validar no Host
ls app-dev
cat app-dev/container.txt

Explicação Técnica — Cenário 3
·Bind Mount conecta diretamente uma pasta do host ao container. 
·Alterações no host aparecem no container em tempo real. 
·Muito utilizado em desenvolvimento. 

Cenário 4 — Compartilhamento Entre Containers
1. Criar Volume Compartilhado
docker volume create shared-data

2. Criar Container Produtor
docker run -d \
--name produtor \
-v shared-data:/dados \
ubuntu \
bash -c "while true; do date >> /dados/log.txt; sleep 5; done"

3. Criar Container Consumidor
docker run -it \
--name consumidor \
-v shared-data:/dados \
ubuntu bash

4. Validar Compartilhamento
Dentro do container consumidor:
cd /dados
cat log.txt
Atualizar:
tail -f log.txt

Explicação Técnica — Cenário 4
·Dois containers compartilham o mesmo volume. 
·O produtor grava dados. 
·O consumidor acessa os dados em tempo real. 

Cenário 5 — Automação de Backup
scripts/backup.sh
#!/bin/bash

DATA=$(date +%Y-%m-%d_%H-%M-%S)

mkdir -p backups

docker exec mysql-prod \
mysqldump -u root --password=123456 empresa \
> backups/empresa_$DATA.sql

tar -czvf backups/mysql_backup_$DATA.tar.gz backups/*.sql

echo "Backup realizado com sucesso."

Dar Permissão
chmod +x scripts/backup.sh

Executar Script
./scripts/backup.sh

Script de Restauração — scripts/restore.sh
#!/bin/bash

docker exec -i mysql-prod \
mysql -u root --password=123456 empresa \
< backups/empresa.sql

echo "Restauração concluída."

Script SQL — scripts/mysql-init.sql
CREATE DATABASE empresa;

USE empresa;

CREATE TABLE usuarios (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nome VARCHAR(100),
    email VARCHAR(100)
);

INSERT INTO usuarios (nome, email)
VALUES
('Joao', 'joao@email.com'),
('Maria', 'maria@email.com'),
('Carlos', 'carlos@email.com');


1. Introdução
Explique:
·Persistência em containers
Persistência em containers é o mecanismo que permite que os dados continuem existindo mesmo depois que um container é parado, reiniciado ou removido.
Por padrão, containers Docker são efêmeros. Isso significa que tudo que é gravado dentro do sistema de arquivos interno do container pode ser perdido quando ele deixa de existir. 
·Containers stateless e stateful
 Os termos stateless e stateful descrevem como uma aplicação lida com dados e estado durante sua execução em containers.
·Importância dos Docker Volumes 
Os Docker Volumes são mecanismos de armazenamento persistente usados para manter dados seguros e independentes do ciclo de vida dos containers.
Eles são fundamentais porque containers Docker são, por natureza, efêmeros.
Ou seja:
containers podem ser destruídos; 
recriados automaticamente; 
atualizados constantemente; 
e seus dados internos podem desaparecer. 
Os volumes resolvem esse problema.
 

2. Ambiente Utilizado
SO: Ubuntu 24.04 LTS
Docker: 28.x
Docker Compose: v2.x
Memória RAM: 8GB
CPU: Intel i5
 
4. Evidências
·docker ps 
·docker volume ls 
·SELECT no MySQL 
·arquivos criados 
·execução dos scripts 
·restauração dos backups 

5. Problemas Encontrados
·conflito de porta 3306 
·permissão de scripts 
·erro de autenticação MySQL 
·container encerrando inesperadamente 

Sugestão de Commits Git
git init

git add .
git commit -m "Estrutura inicial do projeto"

git commit -m "Implementacao do cenario 1"

git commit -m "Implementacao do cenario 2"

git commit -m "Adicionado backup automatizado"

git commit -m "Documentacao final README"

Comandos Úteis para Troubleshooting
Ver Logs
docker logs mysql-prod
Ver Volumes
docker volume inspect mysql-prod-data
Ver Containers
docker ps -a
Entrar no Container
docker exec -it mysql-prod bash

Conclusão Técnica
Nesta atividade foram implementados mecanismos fundamentais de persistência de dados em ambientes containerizados utilizando Docker. Foram explorados conceitos de armazenamento persistente com volumes nomeados, Bind Mounts, compartilhamento de dados entre containers, estratégias de backup/restauração e automação operacional via scripts Bash.
Os testes demonstraram como containers podem ser reconstruídos sem perda de dados quando volumes persistentes são utilizados corretamente, aproximando o ambiente acadêmico de práticas reais utilizadas em ambientes DevOps, Cloud Computing e Infraestrutura Moderna.
