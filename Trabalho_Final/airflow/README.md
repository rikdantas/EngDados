- Primeiro passo: Preparando o .env

        echo -e "AIRFLOW_UID=$(id -u)" > .env


- Segundo passo: Inicializar os bancos de dados

        docker compose up airflow-init
- Terceiro passo: Executando o airflow

        docker compose up
