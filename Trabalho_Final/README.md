# Trabalho final de Engenharia de Dados
Alunos: Paulo Ricardo Dantas e Vínicius Yan Tavares do Nascimento

## Descrição
Pretende-se que seja criada uma solução de processamento de dados em tempo real utilizando Apache Spark Streaming e Apache Airflow, que deverá consumir dados em tempo-real a partir de um Data Lake. O Data Lake deve conter dados estruturados ou semi-estruturados armazenados em bancos de dados PostgreSQL ou MongoDB, bem como arquivos json e csv no sistema de arquivos local. As aplicações Spark deverão ser desenvolvidas em pySpark e deverão consumir os dados em tempo real a partir do Data Lake e realizar transformações e análises dos dados. O Apache Kafka deverá ser utilizado para ingestão e entrega de dados em tempo real para as aplicações Spark. A Figura 1 apresenta a arquitetura sugerida para o projeto. O Apache Airflow será utilizado para orquestrar a programação e o monitoramento dos fluxos de ETL do projeto, garantindo a execução confiável e escalável das tarefas de processamento de dados em tempo real.

## Objetivos 
- Criação de um Data Lake combinando diversas fontes de dados para consumo por parte das aplicações Apache Spark a serem desenvolvidas. A escolha do conjunto de dados (dataset) é livre.
- Criação de fluxos de processamento de dados (streaming) para consumir e processar dados em tempo real com Apache Spark a partir das fontes de dados existentes no Data Lake, com agendamento e orquestração feitos pelo Apache Airflow.
- Realizar uma análise simplificada dos dados que demonstrem o funcionamento dos pipelines criados, com o suporte do Apache Airflow para automatizar a geração e a entrega dessas análises.
- (Opcional) Criação de fluxos de migração de dados dos bancos de dados PostgreSQL para MongoDB e vice-versa, por meio de aplicações Apache Spark Streaming, integradas com o Apache Airflow para agendar e monitorar esses fluxos de migração.


# Resolução
## Preparação da infraestrutura
Para preparar a infraestrutura do projeto foi utilizado containers Docker. Foi utilizado a imagem do airflow para subir o container do mesmo, além de mais 3 containers para o hadoop/spark(um para o mestre e dois para os escravos). O arquivo de Docker Compose usado para subir o container do spark está disponível nesse link: <https://github.com/cmdviegas/hadoop-spark>.

Para fins de reprodução, basta entrar nas pastas correspondentes a cada uma das ferramentas aqui no repositório e seguir o comando para criar os containers.

## Criação do data lake
Será utilizado o dataset do Kaggle: "Paris 2024 Olympic Summer Games", mais especificamente o arquivo athletes.csv. O mesmo pode ser encontrado no link a seguir <https://www.kaggle.com/datasets/piterfm/paris-2024-olympic-summer-games>.

A fim de atender os requisitos do projeto, o dataset vai ser divido em dois: uma parte vai ficar como arquivo CSV e outra parte vai ficar armazenado no banco de dados PostGres.

Para isso, vamos utilizar o pySpark para nos auxiliar a fazer esse trabalho.

### Utilizando pyspark para separar CSV
Com o cluster do spark rodando, primeiro é preciso colocar o arquivo csv dentro do hdfs e para isso foi usado os comandos:

    # Colocando o arquivo do computador local para o container docker
    docker cp athletes.csv node-master:/home/spark

    # Copiando do container para HDFS
    hdfs dfs -put athletes.csv

Com o csv no sistema de arquivos, então podemos começar a usar o spark para manipular o csv:

    # Importando o csv para DataFrame
    df = spark.read.csv("hdfs://node-master:9000/user/spark/athletes.csv", header=True, inferSchema=True)

#### Pré-processamento dos dados
Antes de continuar com a parte de separar, vamos reduzir algumas colunas do dataset que possuam muitos dados nulos ou que possuam informações desnecessárias para o projeto.

    # Escolhendo as colunas para remover
    colunas_a_excluir = ["function", "country_code", "country_long", "nationality_full", "events", "birth_date", "birth_place", "birth_country", "residence_place", "residence_country", "nickname", "hobbies", "occupation", "education", "family", "coach", "reason", "hero", "influence", "philosophy", "sporting_relatives", "ritual", "other_sports"]

    # Criando novo DataFrame sem as colunas
    df_reduzido = df.drop(*colunas_a_excluir)

#### Separando os atletas pelo país que competiu
    # Separando os atletas do Brasil em um dataframe
    df_brazil = df_reduzido.filter(df_reduzido["country"] == "Brazil")

    # Separando os outros atletas em um dataframe
    df_restante = df_reduzido.filter(df_reduzido["country"] != "Brazil")

#### Salvando os dataframes em novos CSVs

    # Salvando o DataFrame dos atletas do Brasil
    df_brazil.coalesce(1).write.csv("/user/spark/brazil_athletes", header=True, sep=";")

    # Salvando o DataFrame dos atletas restantes
    df_restante.coalesce(1).write.csv("/user/spark/other_athletes", header=True, sep=";")

Como estamos usando o HDFS, o coalesce(1) serve para que o arquivo resultante esteja em apenas uma parte. Essas partes vão ser renomeadas como brazil_athletes.csv e other_athletes.csv respectivamente. Após salvar esses CSVs, vamos mover e renomear os mesmos, para fins de melhor tratamento de caminhos.


### Armazenando os atletas brasileiros no postgres
Para fazer isso, iremos usar o driver JDBC assim como foi ensinado durante as aulas para exportar o dataframe para o postgres.

    # Baixando o driver
    wget https://jdbc.postgresql.org/download/postgresql-42.6.0.jar

    # Iniciando o spark com o jar do driver
    pyspark --jars /home/spark/postgresql-42.6.0.jar

    # Importando o dataframe do csv preciamente salvo
    df_brazil = spark.read.csv("hdfs://node-master:9000/user/spark/brazil_athletes.csv", header=True, inferSchema=True, sep=";")

    # Exportando o dataframe para o banco de dados
    df_brazil.write.format("jdbc").option("url","jdbc:postgresql://172.30.0.254:5432/").option("dbtable","atletas_brasil").option("user","postgres").option("password","spark").option("driver","org.postgresql.Driver").save()

## Instalação e configuração do Kafka
Para instalar o Kafka, iremos utilizar o passo a passo também disponibilizado pelo professor durante as aulas. O arquivo para instalar o mesmo vai estar disponícel no repositório como: Exemplos_kafka.txt

### Configuração do kafka

Para configurar o kafka, taambém foi seguido o tutorial para configuração, porém vale lembrar alguns comandos:

    # Mesmo depois de configurado, sempre que iniciar o node-master, precisamos usar os seguintes comandos para carregar o kafka

    # Configurar as variáveis de ambiente no .bashrc
    export KAFKA_HOME="/home/spark/kafka"
    export PATH="$PATH:$KAFKA_HOME/bin"

    # Carregar o .bashrc
    source ~/.bashrc

    # Comando para iniciar o servidor kafka
    kafka-server-start.sh $KAFKA_HOME/config/kraft/server.properties

    # Como criar um tópico
    kafka-topics.sh --create --topic meu-topico --bootstrap-server node-master:9092

### Usando o debezium para monitorar um banco de dados
Iremos utilizar o debezium junto com o kafka para monitorar o banco de dados postgres. Foi seguido o passo a passo do exemplo 3 para realizar essa parte do trabalho:

    # Importando funções
    from pyspark.sql.functions import *
    from pyspark.sql.types import *
    
    # Criar o dataframe do tipo stream, apontando para o servidor kafka e o tópico a ser consumido
    df = (spark.readStream
        .format("kafka")
        .option("kafka.bootstrap.servers", "node-master:9092")
        .option("subscribe", "meu-topico.public.minhatabela2")
        .option("startingOffsets", "earliest")
        .load()
    )

    # Definir o schema dos dados inseridos no tópico
    schema = StructType([
    StructField("payload", StructType([
        StructField("after", StructType([
            StructField("code", IntegerType(), True),
            StructField("name", StringType(), True),
            StructField("name_short", StringType(), True),
            StructField("name_tv", StringType(), True),
            StructField("gender", StringType(), True),
            StructField("country", StringType(), True),
            StructField("country_full", StringType(), True),
            StructField("nationality", StringType(), True),
            StructField("nationality_code", StringType(), True),
            StructField("height", IntegerType(), True),
            StructField("weight", DoubleType(), True),
            StructField("disciplines", StringType(), True),
            StructField("lang", StringType(), True)
            ]))
         ]))
    ])

    # Converter o valor dos dados do Kafka de formato binário para JSON usando a função from_json
    dx = df.select(from_json(df.value.cast("string"), schema).alias("data")).select("data.payload.after.*")

    # Realizar as transformações e operações desejadas no DataFrame 'df'
    # Neste exemplo, apenas vamos imprimir os dados em tela
    ds = (dx.writeStream 
        .outputMode("append") 
        .format("console")
        .option("truncate", False)
        .start()
    )

Com isso, o kafka vai estar monitorando o banco de dados e qualquer dado inserido, vai ser mostrado em tela se o comando anterior estiver sendo executado ainda.

Para ilustrar iremos inserir um dado no postgres como exemplo visual. A saída do último comando mostrado pode ser vista na imagem a seguir:

![](img/monitoramento_antes.png)

Agora, iremos entrar dentro do banco de dados e inserir uma nova linha na tabela atletas_brasil, com o seguinte comando:    

    INSERT INTO atletas_brasil (code, name, name_short, name_tv, gender, country, country_full, nationality, nationality_code, height, weight, disciplines, lang)
    VALUES (123, 'Paulo Ricardo Dantas', 'Paulo Dantas', 'Dantas', 'male', 'Brazil', 'Brazil', 'Brazil', 'BRA', 187, 0.0, 'Basketball', 'Portuguese');

Automaticamente, o terminal em que o Kafka está sendo executado irá mostrar uma nova saída, que vai ser mostrada na imagem a seguir:
![](img/monitoramento_depois.png)

Com isso, terminamos a parte de configuração do kafka.

## Airflow
Devido a dificuldade com a ferramente, não conseguimos implementar o airflow dentro do nosso trabalho