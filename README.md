Este material de estudo foi elaborado para fornecer uma compreensão completa e detalhada dos tópicos essenciais de Otimização de Consultas e Controle de Concorrência, começando do zero, com o objetivo de prepará-lo para a Prova 2. Todos os conceitos técnicos são explicados em linguagem acessível.

***

# GUIA DE ESTUDO COMPLETO PARA PGBD – PROVA 2

## I. Arquitetura e Estruturas de Armazenamento

Para entender a otimização de consultas, precisamos entender como o banco de dados armazena as informações fisicamente.

### 1. Conceitos Básicos de Armazenamento

Quando você acessa dados, o SGBD (Sistema Gerenciador de Banco de Dados) precisa ir ao disco rígido (que é lento) para buscar a informação.

*   **Página / Bloco:** É a menor unidade de dados que o SGBD lê ou escreve no disco. Quando o SGBD precisa de um registro, ele carrega a página inteira que o contém.
*   **Heap:** É uma estrutura de armazenamento desordenada, como um "monte" de registros. Os dados são colocados em qualquer espaço disponível.
    *   **Uso:** O **PostgreSQL** usa *Heaps* para armazenar dados de tabelas. O **MySQL** (engine MyISAM) também usa.
*   **RID (Record ID):** É o endereço físico exato (a localização) de um registro em um arquivo *Heap*. É usado como ponteiro em índices não clusterizados no PostgreSQL.
*   **Por que isso importa?** O principal objetivo de qualquer mecanismo de otimização e indexação é **reduzir o número de páginas de dados a serem carregados para a memória**.

### 2. Tipos e Estruturas de Índices

O índice é um mecanismo que acelera o acesso aos dados, funcionando como um guia telefônico que aponta diretamente para a página correta.

#### 2.1. Tipos de Índices (Hash vs. Ordenado)

| Tipo | Estrutura | Tipos de Acesso Eficiente | Cenário de Uso |
| :--- | :--- | :--- | :--- |
| **Hash** | Usa uma função matemática para mapear a chave para um "balde" (bucket). | **Igualdade** (ótimo desempenho). | Tem **desempenho péssimo em consultas por intervalo**. |
| **Ordenado** | **Árvore B+**. Chaves de busca são armazenadas em ordem. | **Igualdade** e, principalmente, **Intervalo**. | É o índice preferido da maioria dos SGBDs. |

#### 2.2. O Mecanismo da Árvore B+

A Árvore B+ é a estrutura mais comum para índices ordenados e armazena pares `<chave, valor>`.

*   **Busca por Igualdade (Ex: `idM = 9`):** A busca começa na raiz e segue os nós internos, levando **diretamente ao nó folha** que contém a entrada.
*   **Busca por Intervalo (Ex: `idM >= 27`):** A busca localiza o primeiro nó folha que satisfaz a condição. A partir daí, o SGBD **segue as ligações sequenciais** entre os nós folha, lendo as entradas seguintes até o fim do intervalo.

#### 2.3. Índices Compostos (Múltiplas Colunas)

Um índice composto é criado sobre mais de um atributo (Ex: `(cast_order, c_name)`).

*   **Regra de Ouro:** A eficiência depende da **ordem das colunas no índice**. A busca é eficiente se a consulta utilizar filtros nos primeiros níveis do índice.
*   **Melhor Cenário:** Busca por **igualdade em todos os níveis** (Ex: `cast_order = 1 AND c_name = 'Jack'`).
*   **Cenário Ineficiente:**
    *   **Pular Nível:** Se o filtro ignora o primeiro nível (Ex: `c_name = 'Jack'` sem filtrar `cast_order`), o índice não pode ser usado eficientemente e pode levar a varredura completa.
    *   **Filtro Disjuntivo (`OR`):** Geralmente, exige a leitura de toda a tabela, pois o `OR` impede o uso eficiente da ordenação (Ex: `cast_order = 1 OR c_name = 'Jack'`).

#### 2.4. Índices Clusterizados vs. Não Clusterizados

| Característica | Índice Clusterizado (Primário) | Índice Não Clusterizado (Secundário) |
| :--- | :--- | :--- |
| **Conteúdo do Nó Folha** | **Armazena o registro de dados completo**. | Armazena a chave de busca e um **ponteiro** para o registro. |
| **Organização da Tabela** | Define a ordem física dos dados. | Não define a ordem física. |
| **Ponteiro (MySQL InnoDB)** | N/A | Ponteiro é a **chave primária** do registro. |
| **Ponteiro (PostgreSQL Heap)** | N/A | Ponteiro é o **RID** (Record ID). |
| **Complementação:** | Nenhuma. | Necessária! Após buscar no índice, o SGBD usa o ponteiro (PK ou RID) para acessar o registro completo. |

### 3. Critérios de Seletividade (Quando Indexar?)

Não vale a pena indexar todos os atributos. O critério fundamental é a **Seletividade**.

*   **O que é Seletividade?** É a quantidade de registros que se espera recuperar a cada consulta.
    *   **Alta Seletividade:** O filtro retorna **poucos registros** (Ex: `Nome = 'Fulano'`). É altamente vantajoso usar um índice neste caso.
    *   **Baixa Seletividade:** O filtro retorna **muitos registros** (Ex: `Nível = 'A1'`, que representa 20% do arquivo). O otimizador pode decidir que é **mais eficiente fazer uma varredura completa da tabela** (*Table Scan*) do que ler o índice e fazer acessos não sequenciais para recuperar os 20% dos registros espalhados no disco.
*   **Outros Critérios:** Indexar atributos usados em `JOINs`, `WHERE`, `ORDER BY` e `GROUP BY`.

***

## II. Otimização e Planos de Execução

Um **Plano de Execução** é a "receita" que o SGBD cria para rodar a consulta, definindo a **ordem das operações** e os **algoritmos** a serem usados.

### 1. Componentes do Plano (Arquitetura Volcano)

Os planos são árvores de operadores (ou nós) onde os registros fluem de baixo para cima (`Volcano Architecture`).

| Métrica no `EXPLAIN ANALYZE` | Significado Simples |
| :--- | :--- |
| **`cost`** | Custo estimado da operação. O SGBD tenta escolher o plano de menor custo. |
| **`rows`** | Quantidade estimada de registros a serem retornados (antes da execução). |
| **`actual time`** | Tempo real gasto (em milissegundos). O segundo valor é o tempo total. |
| **`actual rows`** | Quantidade real de registros recuperados. |

### 2. Operadores de Junção

#### 2.1. Nested Loop Join (NLJ)

*   **Funcionamento:** Para cada registro do **lado externo** (esquerda), o SGBD busca as correspondências no **lado interno** (direita).
*   **Eficiência:** É altamente eficiente quando o **lado interno está indexado** (usando `Index Lookup` ou `Key Lookup`).
*   **Custo:** Baixo consumo de memória, pois os registros fluem por *pipeline*.

#### 2.2. Hash Join (HJ)

*   **Funcionamento:**
    1.  **Construção:** Cria uma tabela *Hash* temporária na memória sobre o lado interno (é uma operação **materializada**, consumindo memória).
    2.  **Sonda:** Para cada registro do lado externo, busca rapidamente as correspondências na tabela *Hash*.
*   **Eficiência:** Principalmente usado quando **não existem índices** sobre as colunas de junção. Evita a necessidade de varrer a tabela interna repetidamente.
*   **Decisão do Otimizador:** O SGBD usa Hash Join em consultas **pouco seletivas** ou quando não há índice, enquanto prefere NLJ para consultas seletivas com índices.

### 3. Otimizações Estruturais de Consultas (SQL)

O desenvolvedor pode escrever consultas para forçar o otimizador a escolher um caminho mais rápido.

#### 3.1. Uso de Colunas e Índices de Cobertura (`Covering Index`)

*   **Evitar `SELECT *`:** Exige que **todas as colunas** sejam recuperadas, o que aumenta o custo de memória e **impede o uso de um *Covering Index***.
*   **Otimizar com *Covering Index*:** Se você solicitar apenas colunas que **já estão presentes no índice** (índice de cobertura), o SGBD resolve a consulta lendo apenas o índice.
    *   *Exemplo:* `SELECT release_year FROM movie WHERE release_year > 1950` é resolvida apenas lendo o índice `Idx_year(year, idM)`, sem precisar acessar a tabela `movie`.

#### 3.2. Uso do `COUNT(*)`

*   **`COUNT(*)` vs. `COUNT(col)`:** `COUNT(*)` é mais rápido porque apenas conta os registros, **sem precisar verificar a presença de valores nulos** na coluna especificada.
*   **Sintaxe Otimizada:** Use `COUNT(*)` ou, se realmente precisar contar não nulos, use `COUNT(*) WHERE coluna IS NOT NULL`.

#### 3.3. Evitar Duplicatas Desnecessárias (`DISTINCT` e `UNION`)

*   **O Custo da Deduplicação:** O SGBD, para garantir a unicidade (`DISTINCT` ou `UNION`), usa uma operação custosa de **ordenação seguida de remoção de registros idênticos**. Isso geralmente exige o uso de uma **tabela temporária** (`tmp table` ou `Temporary table with deduplication`).
*   **Otimização:** Se você souber que o resultado não terá duplicatas, use:
    *   `SELECT ALL` em vez de `SELECT DISTINCT`.
    *   `UNION ALL` em vez de `UNION`.

#### 3.4. Uso do `LIMIT`

O `LIMIT` é crucial para otimizar consultas que envolvem `ORDER BY`.

*   **Redução do Custo de Ordenação:** Permite que o SGBD use algoritmos de **ordenação parcial** (como `Top-N sort` ou `top-N heapsort`). Esses algoritmos ordenam apenas os N registros necessários, evitando a ordenação completa, o que economiza tempo e evita o uso de ordenação em disco (`external merge`).

#### 3.5. Ordem das Junções e Isolamento

*   **Forçar Ordem (`STRAIGHT_JOIN`):** A ordem das junções é tipicamente definida pelo SGBD. Se as estatísticas internas estiverem erradas, o desenvolvedor pode usar `STRAIGHT_JOIN` (MySQL) para forçar a ordem que considera mais eficiente, geralmente começando pelas junções que geram o menor resultado intermediário.
*   **Isolar Trechos Independentes (`OR` vs. `UNION`):** Evite usar o conector `OR` em filtros que dependem de diferentes junções. O `OR` pode impedir o uso de índices e levar à criação de uma tabela intermediária enorme.
    *   **Otimização:** Use **`UNION`** para isolar cada filtro em um `SELECT` separado. Isso permite que os índices sejam usados em cada parte e reduz drasticamente o volume de dados intermediário.

***

## III. Estatísticas para Estimativa de Custo

O Otimizador de Consultas usa **estatísticas** para prever o número de registros que cada operação irá retornar (sua **seletividade**), o que é a base para calcular o custo e escolher o plano.

### 1. Estatísticas Fundamentais

*   $\mathbf{n(r)}$: Número de tuplas (registros) na relação $r$ (tabela).
*   $\mathbf{v(A, r)}$: **Cardinalidade** do atributo $A$, ou seja, o número de valores **distintos** em $A$.
*   $\mathbf{min(A, r)}$ e $\mathbf{max(A, r)}$: Menor e maior valor para o atributo $A$.

### 2. Estimativa com Filtros (Seleção)

A seletividade $P(A)$ é a probabilidade de um registro satisfazer o filtro.

*   **Filtro por Igualdade (`WHERE tab.A = valor`):**
    $$P(A) = \frac{1}{v(A, \text{tab})} \text{}$$
    $$\text{Estimativa} = n(tab) \cdot P(A) \text{}$$
    *Exemplo: $n(\text{proj})=2.000$ e $v(\text{custo}, \text{proj})=50$. $\text{Estimativa} = 2.000 \cdot (1/50) = 40$ registros.

*   **Filtro por Faixa (`WHERE tab.A <= v`):** Se os valores min e max são conhecidos:
    $$P(A) = \frac{v - \min(A, tab) + 1}{\max(A, tab) - \min(A, tab) + 1} \text{}$$
    Se não há informações estatísticas para faixa, usa-se $P(A) = 1/2$.

*   **Negação (`NOT`):** A probabilidade de um evento não acontecer é o complemento:
    $$P(\bar{A}) = 1 - P(A) \text{}$$

*   **Filtro por Conjunção (`AND`):** Interseção de eventos independentes, multiplicando as probabilidades:
    $$P(A \cap B) = P(A) \cdot P(B) \text{}$$

*   **Filtro por Disjunção (`OR`):** União de eventos, somando probabilidades e subtraindo a interseção:
    $$P(A \cup B) = P(A) + P(B) - P(A \cap B) \text{}$$
    Se os filtros são mutuamente exclusivos (não podem ocorrer juntos), $P(A \cap B)$ é zero, simplificando a fórmula.

### 3. Estimativa com Junções

*   **Junção Chave Primária/Chave Estrangeira (PK/FK):**
    A estimativa é dada pelo número de registros na tabela da Chave Estrangeira (FK) multiplicado pela seletividade dos filtros aplicados na tabela da Chave Primária (PK). O máximo de registros gerados é $n(FK)$.
    $$\text{Estimativa} = n(FK) \cdot \text{sel}(PK) \text{}$$
    *Exemplo:* Junção entre `proj` (PK) e `aloc` (FK). $n(\text{aloc})=100.000$. Filtro: `custo = 30.000` em `proj`. $v(\text{custo}, \text{proj})=10$. $\text{Estimativa} = 100.000 \cdot (1/10) = \mathbf{10.000}$ registros.

*   **Junção Sem Critério (`CROSS JOIN`):** A estimativa é o produto da quantidade de registros que chegam em cada lado da junção.
    $$\text{Estimativa} = n(tab1) \cdot n(tab2) \text{}$$

***

## IV. Junções Avançadas e SQL Complexo

### 1. Semi-junção (`IN` / `EXISTS`)

*   **Definição:** Retorna apenas as tuplas da **parte principal** que possuem **pelo menos uma correspondência** na parte secundária. Cada tupla é retornada uma única vez.
*   **Sintaxe SQL:** Usa `EXISTS` (subconsulta correlacionada) ou `IN` (subconsulta independente).
    ```sql
    SELECT title FROM movie m WHERE EXISTS (SELECT 1 FROM movie_cast mc WHERE m.idM = mc.idM)
    ```
*   **Representação Interna:** O SGBD usa `Nested Loop Left Semi Join` ou `Hash Left Semi Join`. O algoritmo **apenas verifica a existência** da correspondência, sem realizar o cruzamento.
*   **Eficiência vs. Junção Regular:** Semi-junção é **mais eficiente** do que simular com `INNER JOIN` + `DISTINCT`, pois evita o **cruzamento sobressalente** e o *overhead* de **remoção de duplicatas** (`Hash Duplicate Removal`).

### 2. Anti-junção (`NOT IN` / `NOT EXISTS`)

*   **Definição:** Retorna apenas as tuplas da **parte principal** que **não possuem correspondência** na parte secundária.
*   **Sintaxe SQL:** Usa `NOT EXISTS` ou `NOT IN`.
    ```sql
    SELECT title FROM movie m WHERE NOT EXISTS (SELECT 1 FROM movie_cast mc WHERE m.idM = mc.idM)
    ```
*   **Representação Interna:** O SGBD usa `Nested Loop Left Anti Join` ou `Hash Left Anti Join`. O algoritmo **apenas verifica a não existência**.
*   **Eficiência vs. Junção Externa:** Anti-junção é **melhor** do que simular com `LEFT JOIN` + `WHERE IS NULL` porque evita o trabalho de localizar os cruzamentos e depois descartá-los.

#### `NOT IN` vs. `NOT EXISTS`

*   **Preferência:** Use **`NOT EXISTS`**.
*   **Risco do `NOT IN`:** Se a subconsulta usada com `NOT IN` retornar **qualquer valor nulo**, a consulta inteira retornará um conjunto vazio, pois qualquer comparação com `NULL` resulta em "incerto" (`UNKNOWN`), e resultados incertos nunca são retornados. O `NOT EXISTS` não sofre desse problema.

### 3. Common Table Expression (CTE)

*   **Definição:** É uma expressão de tabela temporária definida pela cláusula `WITH` dentro de uma consulta `SELECT`, `INSERT`, `UPDATE` ou `DELETE`.
*   **Benefícios:** Melhora a **legibilidade**, permite a **reutilização** de subconsultas complexas, e pode ser **mais eficiente** do que usar *Views* (que são permanentes) ou repetir a subconsulta.
*   **Sintaxe:**
    ```sql
    WITH media_custo AS (SELECT AVG(custo) AS media FROM proj)
    SELECT p.* FROM proj p JOIN media_custo m ON p.custo > m.media;
    ```
*   **CTEs Recursivas:** Usam `WITH RECURSIVE` para resolver cenários hierárquicos (como a subordinação de funcionários), onde a CTE se chama dentro de si mesma.

### 4. Funções de Janela (`Window Functions`)

Permitem executar cálculos (como ranqueamento ou agregação) sobre um conjunto de linhas relacionadas (**janela**), **sem agrupar os dados**.

*   **Estrutura:** A cláusula `OVER` define a janela.
    *   `PARTITION BY`: Divide os dados em grupos lógicos (janelas).
    *   `ORDER BY`: Define a ordem dentro da janela (necessário para cálculos acumulados ou rankings).
*   **Exemplos de Funções:**
    *   **Ranqueamento:** `ROW_NUMBER()`, `RANK()` (deixa "buracos" em caso de empate), `DENSE_RANK()` (não deixa buracos).
    *   **Agregadas Acumuladas:** `SUM()` ou `AVG()`. Se usadas com `ORDER BY` na cláusula `OVER`, a `SUM()` calcula o total **acumulado**.

### 5. OLAP (Online Analytical Processing)

*   **Contexto:** OLAP é usado em **Data Warehouses**, sistemas projetados para análise de grandes volumes de dados históricos (suporte à decisão), diferente dos bancos operacionais (OLTP).
*   **Fatos e Dimensões:** Dados são organizados em Dimensões (categorias, ex: Lider, Duração, Status) e Fatos (métricas numéricas associadas, ex: Custo).
*   **Operadores OLAP:**
    *   **Roll Up:** Agrega informações detalhadas em um resumo (Ex: Lider $\rightarrow$ Sexo).
    *   **Drill Down:** Adiciona detalhes a uma agregação.
    *   **Slice:** Escolhe um **único valor** de uma dimensão (Ex: `WHERE duracao = 24`).
    *   **Dice:** Forma um subcubo, definindo **múltiplos valores** em uma ou mais dimensões (Ex: `WHERE duracao IN (24, 36)`).
    *   **Pivot:** Aplica uma rotação nas dimensões (transforma linhas em colunas). No SQL, é frequentemente simulado usando a cláusula `CASE` dentro de uma função `SUM`.

***

## V. Controle de Concorrência e Transações

### 1. Transações e Propriedades ACID

Uma transação é uma unidade lógica de trabalho que executa um conjunto de operações (`READ` e `WRITE`).

*   **Estados:** **Active** (executando) $\rightarrow$ **Partially Committed** (último comando executado) $\rightarrow$ **Committed** (sucesso) ou **Failed** $\rightarrow$ **Aborted** (revertida).
*   **Sintaxe SQL:** Usada `START TRANSACTION`, finalizada por `COMMIT` ou `ROLLBACK`.

As transações devem satisfazer as propriedades **ACID** para garantir a integridade dos dados:

| Propriedade | Significado (Simples) | O que Garante |
| :--- | :--- | :--- |
| **Atomicidade** | **"Tudo ou nada."** Ou todas as operações são executadas, ou nenhuma é. | O `ROLLBACK` é usado para reverter uma transação abortada e manter o estado original. |
| **Consistência** | Leva o banco de um estado correto para outro (mantendo as regras de negócio). | Evita resultados ilógicos (Ex: Retirar 50 e depositar 500). |
| **Isolamento** | Transações concorrentes devem parecer executadas **serialmente** (uma após a outra), mesmo quando intercaladas. | Impede que uma transação veja o banco em um estado parcialmente atualizado por outra. |
| **Durabilidade** | Após o `COMMIT`, as mudanças devem persistir permanentemente, mesmo em caso de falhas. | Garante que o usuário não perca a atualização confirmada. |

### 2. Schedules e Concorrência

Um *schedule* é a ordem cronológica das instruções de transações concorrentes.

*   **Schedule Serial:** Transações executadas uma após a outra, sem intercalação. É sempre consistente, mas lento.
*   **Schedule Concorrente:** Instruções de transações são executadas de forma intercalada. É mais eficiente (maior *throughput* e menor tempo de resposta).

O protocolo de concorrência deve gerar schedules que sejam:

1.  **Recuperáveis:** Se uma transação ($T_9$) lê um item escrito por outra ($T_8$), o **commit de $T_8$ deve aparecer antes do commit de $T_9$**. Caso $T_8$ aborte, $T_9$ deve ser capaz de reverter.
2.  **Sem Rollback em Cascata:** Se $T_A$ lê um item escrito por $T_B$, o **commit de $T_B$ deve ocorrer antes que $T_A$ tenha feito a leitura**. Isso é desejável, pois reverter uma transação que já foi lida por outras gera um efeito cascata de reversões, que é custoso. **Todo schedule sem cascata é recuperável**.

### 3. Protocolos Baseados em Lock (Bloqueio)

Um lock (bloqueio) é o mecanismo usado para controlar o acesso concorrente a um item de dados.

*   **Lock Compartilhado (S):** Permite apenas **leitura**. Várias transações podem ter lock S sobre o mesmo item (compatíveis).
*   **Lock Exclusivo (X):** Permite **leitura e escrita**. Se uma transação tem lock X, nenhuma outra transação pode obter qualquer lock sobre o item (incompatível).

#### Protocolo Two-Phase Locking (2PL)

O 2PL exige que a transação tenha duas fases:
1.  **Fase de Crescimento:** A transação só pode **obter locks**.
2.  **Fase de Encolhimento:** A transação só pode **liberar locks**.

**Problemas do 2PL Puro:** O 2PL puro tem inconvenientes, pois **não previne rollbacks em cascata** e **gera schedules não recuperáveis**.

#### Protocolo Rigorous Two-Phase Locking (2PL Rigoroso)

Esta é a versão mais utilizada e robusta.
*   **Regra:** A transação deve **segurar todos os locks** (S ou X) **até a instrução final de `COMMIT` ou `ABORT`**.
*   **Vantagens:** Previna rollbacks em cascata e gera somente schedules recuperáveis. Os locks são realizados automaticamente.

### 4. Deadlocks, Starvation e MVCC

*   **Deadlock:** Ocorre quando um ciclo de espera se forma (Ex: T1 espera por um recurso de T2, e T2 espera por um recurso de T1). Nenhuma transação pode prosseguir.
    *   **Tratamento:** Deadlocks são considerados um "mal necessário". São resolvidos através de um algoritmo de **detecção** (usando um **Grafo de Espera** - *wait-for graph*) que identifica o ciclo e **aborta** uma das transações envolvidas, liberando seus locks.
    *   **Prevenção:** Técnicas de prevenção (timeout, pré-declaração de locks) são caras e raramente usadas. O MySQL usa timeout como estratégia de prevenção.

*   **Starvation:** Ocorre quando uma transação espera muito tempo por um item, pois a prioridade é dada a outras transações (geralmente de escrita).

*   **Protocolos Multiversionados (MVCC):** São protocolos **menos bloqueantes**, baseados em múltiplas versões de um registro.
    *   **Vantagem:** **Leitura não bloqueia escrita** e **escrita não bloqueia leitura**.

### 5. Controle de Concorrência no MySQL (InnoDB vs. MyISAM)

O comportamento de concorrência depende da *engine* do MySQL.

| Característica | InnoDB | MyISAM |
| :--- | :--- | :--- |
| **Transações / ACID** | Suporta transações e garantias ACID. | Não suporta transações nem garantias ACID. |
| **Tipo de Bloqueio** | Bloqueio de **Registro** (*Row level locking*). | Bloqueio de **Tabela** (*Table level locking*). |
| **Deadlocks** | Podem ocorrer, resolvidos por detecção e aborto. | Não são possíveis. |
| **Starvation** | Geralmente evitado (a menos que o programador force a prioridade). | Pode ocorrer, pois escritas têm prioridade sobre leituras. |

**Dicas para Otimizar o InnoDB e Reduzir Deadlocks:**
1.  **`SELECT ... FOR UPDATE`:** Adquire **bloqueios exclusivos antecipadamente** para os registros lidos, garantindo que não sejam alterados antes do `COMMIT`.
2.  **Ordenar Solicitações:** Pedir bloqueios (`SELECT ... FOR UPDATE`) sempre na mesma ordem (Ex: `aloc`, depois `func`, depois `proj`) para evitar ciclos de espera.
3.  **Indexar Colunas de Filtro:** Se a coluna de filtro for indexada (Ex: `WHERE custo = 0`), apenas os registros relevantes são bloqueados, o que **aumenta a concorrência** e reduz as chances de *deadlock*.

**Problemas do MyISAM:** O bloqueio de tabela reduz o tempo médio de resposta, pois comandos rápidos de leitura ficam presos atrás de comandos demorados. Isso pode gerar *starvation* para leituras.
