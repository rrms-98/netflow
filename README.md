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
* Facilidade de retenção e limpeza de dados antigos
