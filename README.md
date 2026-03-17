# netflow
CONSULTA À LOGS DE CGNAT
Sistema de Banco de Dados CGNAT – NetFlow

Descrição Geral:

O banco de dados "netflow" é responsável pelo armazenamento de logs de tradução NAT (CGNAT), permitindo rastreamento de sessões de usuários conforme exigências operacionais e legais.

Estrutura:

Tabela principal: flows

A tabela flows é particionada por intervalo de tempo (mensal), permitindo alta performance e escalabilidade.

Particionamento:

* Cada partição representa um mês de dados
* Exemplo:

  * flows_2025_01
  * flows_2025_02
  * flows_2026_03

Campos:

* time (timestamp): data/hora do evento
* event (text): tipo do evento (ADD ou DELETE)
* src_ip (inet): IP privado do cliente
* nat_ip (inet): IP público utilizado
* port_start (integer): início do range de portas
* port_end (integer): fim do range de portas

Índices:

Foi implementado índice global:

(nat_ip, port_start, port_end, time)

Finalidade:

Permitir consultas rápidas para identificar qual cliente utilizava determinado IP público e porta em um intervalo de tempo específico.

Extensões:

* btree_gist: suporte a indexação avançada

Permissões:

* Usuário "netflow" possui acesso total às tabelas
* Usuário "postgres" possui controle administrativo

Benefícios da arquitetura:

* Alta performance em consultas
* Escalabilidade para grandes volumes de dados
* Organização eficiente por data
* Facilidade de retenção e limpeza de dados 

INSTALAÇANDO DEPENDENCIAS:

POSTGRE
```
apt update
apt install postgresql -y
```
CRIANDO USUARIO E BANCO:
```
sudo -u postgres psql
CREATE USER netflow WITH PASSWORD 'senha_forte';
CREATE DATABASE netflow OWNER netflow;
```

QUERY PARA CRIAÇÃO DE PARTIÇÃO DINÂMICA DENTRO DA TABELA FLOWS:
```
CREATE OR REPLACE FUNCTION create_partition(month_start date)
RETURNS void AS $$
DECLARE
    month_end date;
    table_name text;
BEGIN
    month_end := month_start + interval '1 month';
    table_name := 'flows_' || to_char(month_start, 'YYYY_MM');

    EXECUTE format(
        'CREATE TABLE IF NOT EXISTS %I PARTITION OF flows
         FOR VALUES FROM (%L) TO (%L)',
        table_name, month_start, month_end
    );
END;
$$ LANGUAGE plpgsql;
```

CRIANDO PARTIÇÃO:
```
SELECT create_partition('2026-04-01');
```
AUTOMATIZAR COM O CRONTAB:
```
crontab -e
0 0 1 * * psql -U postgres -d netflow -c "SELECT create_partition(date_trunc('month', now())::date);"
```
QUERY PARA REALIZAR O DELETE NA TABELA QUE TEM 13 MESES:
```
DO $$
DECLARE
    part RECORD;
    limite DATE := date_trunc('month', now()) - interval '13 months';
BEGIN
    FOR part IN
        SELECT inhrelid::regclass AS tabela
        FROM pg_inherits
        WHERE inhparent = 'public.flows'::regclass
    LOOP
        IF substring(part.tabela::text from 'flows_(\d{4}_\d{2})') IS NOT NULL THEN
            IF to_date(substring(part.tabela::text from '(\d{4}_\d{2})'), 'YYYY_MM') < limite THEN
                EXECUTE 'DROP TABLE ' || part.tabela;
            END IF;
        END IF;
    END LOOP;
END $$;
```
AUTOMAÇÃO DA QUERY:
```
crontab -e
```
```
DO $$
DECLARE
    part RECORD;
    limite DATE := date_trunc('month', now()) - interval '13 months';
BEGIN
    FOR part IN
        SELECT inhrelid::regclass AS tabela
        FROM pg_inherits
        WHERE inhparent = 'public.flows'::regclass
    LOOP
        IF substring(part.tabela::text from 'flows_(\d{4}_\d{2})') IS NOT NULL THEN
            IF to_date(substring(part.tabela::text from '(\d{4}_\d{2})'), 'YYYY_MM') < limite THEN
                EXECUTE 'DROP TABLE ' || part.tabela;
            END IF;
        END IF;
    END LOOP;
END $$;
```



