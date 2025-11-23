Este material foi elaborado para ser um documento de estudo completo e introdutório sobre os tópicos de otimização de consultas e controle de concorrência, detalhando conceitos, fornecendo sintaxe SQL e explicando os mecanismos internos dos SGBDs.

***

## 1. Fundamentos de Armazenamento e Índices

O acesso a dados em bancos de dados relaciona-se fundamentalmente ao custo de Input/Output (I/O) de disco. Tabelas muito grandes são mantidas no disco, e a meta dos mecanismos de indexação é **reduzir o número de páginas de dados a serem carregados para a memória**.

### Conceitos Básicos de Armazenamento

*   **Página/Bloco:** É a unidade mínima de dados lida ou escrita no disco.
*   **Heap:** É uma estrutura de dados de armazenamento onde os registros são colocados em qualquer espaço disponível, sem ordem (usado pelo PostgreSQL e MySQL MyISAM). Os registros em *Heaps* são localizados pelo seu **RID (Record ID)**, que é o endereço físico onde o registro foi armazenado.

### Tipos de Índices

Existem dois tipos básicos de índices, baseados na forma como as chaves de busca são armazenadas:

| Tipo de Índice | Estrutura | Tipos de Acesso Eficientes | Por que é Usado |
| :--- | :--- | :--- | :--- |
| **Ordenado** | **Árvores B+** (mais comuns). | **Igualdade** e, principalmente, **Intervalo**. | É o índice preferido dos SGBDs devido ao seu bom desempenho geral. |
| **Hash** | Função hash mapeia chaves para *buckets*. | **Igualdade** (desempenho ótimo). | Desempenho é **péssimo em consultas por intervalo**. |

### Diferença entre Índices Clusterizados e Não Clusterizados

A diferença está naquilo que é armazenado no nível folha da Árvore B+:

1.  **Índice Clusterizado (Primário):** A própria árvore B+ **armazena os registros completos**. O nível folha guarda todos os registros, organizados pela chave primária. Uma tabela só pode ter um índice clusterizado (geralmente a chave primária).
    *   **Exemplo (MySQL InnoDB):** O índice primário é clusterizado, e o nível folha dá acesso ao registro completo.
2.  **Índice Não Clusterizado (Secundário):** É uma árvore usada apenas para facilitar o acesso. O nível folha guarda um **ponteiro** para o registro.
    *   **Em SGBDs Clusterizados (MySQL InnoDB):** O ponteiro é a **chave primária** do registro. É necessária uma busca adicional na B+ Tree da chave primária para recuperar o registro completo (complementação).
    *   **Em SGBDs baseados em Heap (PostgreSQL):** O ponteiro é o **RID** (Record ID) do registro.

### Funcionamento da Busca na Árvore B+ (Passo-a-Passo)

A Árvore B+ é a estrutura padrão para índices ordenados.

*   **Busca por Igualdade:** A busca desce pela árvore e leva **diretamente ao nó folha** que contém a chave buscada.
    *   *Exemplo:* `idM = 9`. A busca é guiada até o nó folha que contém `idM=9`.
*   **Busca por Intervalo:** A busca leva ao **primeiro nó folha** que satisfaz o critério. A partir desse ponto, o SGBD lê sequencialmente as entradas seguintes (pois os nós folha são ligados).
    *   *Exemplo:* `idM >= 27`. O sistema localiza `idM = 27` e lê sequencialmente todas as entradas subsequentes.

### Critérios para Criação de Índices

A criação de índices gera sobrecarga na atualização e ocupa espaço. A escolha deve ser criteriosa:

1.  **Foco:** Indexar atributos **muito usados em junções e filtros**. Também pode-se indexar colunas usadas em **`ORDER BY`** ou **`GROUP BY`**.
2.  **Seletividade:** Atributos com **alta seletividade** (que retornam poucos registros quando filtrados) são os melhores candidatos.
    *   **Não Compensa:** Atributos pouco seletivos (ex: se o filtro retornar 20% do arquivo) geralmente levam o otimizador a preferir um *Table Scan* em vez de ler o índice e fazer acessos não sequenciais para recuperar os dados.
3.  **Índices Compostos:** Os filtros devem se aplicar aos primeiros níveis do índice para que ele seja eficaz.
    *   **Melhor Cenário:** Busca por **igualdade em todos os níveis** (ex: `cast_order = 1 AND c_name = 'Jack'`).
    *   **Pior Cenário:** Filtro disjuntivo (`OR`) ou filtro ignorando o primeiro nível do índice (ex: `c_name = 'Jack'` sem filtrar `cast_order`).

***

## 2. Planos de Execução de Consultas

O plano de execução é uma árvore de **operadores** que define o algoritmo e a ordem de execução dos comandos SQL. Os registros fluem dos nós fonte (tabelas/índices) para cima, em direção ao operador final (arquitetura **Volcano**).

### Interpretação do `EXPLAIN ANALYZE` (MySQL/PostgreSQL)

O comando `EXPLAIN ANALYZE` (ou similar) fornece detalhes cruciais sobre o plano escolhido pelo SGBD:

| Métrica | Descrição |
| :--- | :--- |
| **`cost`** | **Custo estimado** da operação. O otimizador escolhe o plano de menor custo. |
| **`rows`** | **Quantidade estimada** de registros. |
| **`actual time`** | **Tempo real** gasto (em milissegundos). O primeiro valor é o tempo para obter o primeiro registro, e o segundo é o tempo total. |
| **`actual rows`** | **Quantidade real** de registros recuperados. |

### Operadores de Filtro e Busca

*   **Scan Sequencial (`Full Table Scan` / `Seq Scan`):** Ocorre quando a tabela é varrida inteiramente, registro por registro. O filtro é aplicado sobre cada registro lido.
    *   **Uso:** É usado quando não há índice ou quando o otimizador decide que o índice não é vantajoso por causa da **baixa seletividade** do filtro (lendo uma porcentagem muito grande do arquivo).
*   **Index Scan (`Index Lookup` / `Index Range Scan`):** O índice é acessado para localizar os registros. No MySQL, `Index Range Scan` é usado para filtros por intervalo.

### Otimização por `Covering Index`

Um índice é de **cobertura** (*covering index*) quando ele **contém todas as colunas necessárias** para resolver a consulta (colunas do `SELECT` e do `WHERE`).

*   **Benefício:** Se o índice for de cobertura, a consulta é resolvida apenas na estrutura do índice. Isso é mais rápido, pois o SGBD **não precisa fazer a complementação** (acessar a tabela principal).
*   **Exemplo:** Um índice `Idx_year(year, idM)` pode cobrir a consulta `SELECT year FROM movie WHERE year > 1950`, pois a coluna `year` está no índice.

### Algoritmos de Junção

| Algoritmo | Nested Loop Join (NLJ) | Hash Join (HJ) |
| :--- | :--- | :--- |
| **Funcionamento** | Para cada registro do lado **externo**, busca as correspondências no lado **interno**. | **Materializa** o lado interno em uma tabela *Hash*. Varre o lado externo e busca na tabela *Hash*. |
| **Vantagem** | Eficiente quando o lado interno **possui índice** sobre a coluna de junção (usando *Key Lookup*). | Eficiente quando **não existe índice** sobre a coluna alvo, pois o *table scan* do lado interno é feito apenas uma vez. |
| **Custo** | Baixo consumo de memória (fluxo por *pipeline*). | **Alto consumo de memória** (operação materializada). |
| **Melhor Cenário** | Consulta **seletiva** com índice. | Consulta **pouco seletiva** sem índice, envolvendo grandes volumes. |

***

## 3. Otimização de Consultas SQL

O desenvolvedor deve tomar cuidados ao criar consultas para auxiliar o otimizador.

### Remover Colunas Desnecessárias (`SELECT *`)

Evitar `SELECT *` e listar apenas as colunas necessárias é fundamental, pois:
*   **Custo:** A recuperação de todas as colunas consome mais espaço em memória e largura de banda.
*   **Índice de Cobertura:** O `SELECT *` exige que o arquivo que contém todas as colunas seja acessado, **impedindo o uso de um *Covering Index***.

### Usar `COUNT(*)`

A função `COUNT(*)` é geralmente **mais rápida** do que `COUNT(col)` porque apenas conta os registros, **sem precisar verificar a presença de valores nulos** na coluna especificada.

**Sintaxe Otimizada:**
```sql
-- Versão menos otimizada (verifica nulos)
SELECT COUNT(nomeProj) FROM proj WHERE custo > 100.000

-- Versão mais otimizada (apenas conta registros)
SELECT COUNT(*) FROM proj WHERE custo > 100.000
```

### Evitar `SELECT DISTINCT` e `UNION`

O uso de `DISTINCT` e `UNION` (que remove duplicatas) é custoso porque exige que o SGBD realize uma **operação de ordenação** seguida de remoção de registros idênticos para garantir a unicidade.

*   Essa operação frequentemente requer uma **tabela temporária** para a deduplicação (`Temporary table with deduplication` ou `Hash Duplicate Removal` em planos).
*   **Otimização:** Se você souber que o resultado já é único (ex: a chave primária está sendo retornada), use **`SELECT ALL`** ou **`UNION ALL`**.

**Exemplo de Otimização (`UNION` vs. `UNION ALL`):**
```sql
-- Versão Lenta: UNION exige ordenação e remoção de duplicatas
SELECT nome FROM func UNION SELECT nome FROM cliente;

-- Versão Otimizada: UNION ALL simplesmente junta os resultados (Append)
SELECT nome FROM func UNION ALL SELECT nome FROM cliente;
```

### Uso do `LIMIT`

O `LIMIT` permite otimizar consultas que usam `ORDER BY`.

*   **Algoritmo Otimizado:** O SGBD pode usar um algoritmo mais eficiente, como o **`Top-N sort`** ou **`top-N heapsort`**, que evita a ordenação total dos registros, processando apenas o suficiente para retornar os N primeiros.

**Exemplo de Sintaxe:**
```sql
-- Ordenação completa (pode usar ordenação externa/disco)
SELECT nome, salario FROM func ORDER BY salario;

-- Ordenação parcial (usa Top-N sort, em memória)
SELECT nome, salario FROM func ORDER BY salario LIMIT 2;
```

### Forçar Ordem das Junções (`STRAIGHT_JOIN`)

Em junções complexas (mais de duas tabelas), o SGBD geralmente escolhe a ordem mais eficiente com base em estatísticas. Contudo, se as estatísticas forem imprecisas, ele pode errar.

*   **Uso:** É mais eficiente realizar primeiro a junção entre as tabelas que resultarão no menor conjunto de registros.
*   **Sintaxe (MySQL):** O desenvolvedor pode usar **`STRAIGHT_JOIN`** para forçar o SGBD a seguir a ordem de junção especificada na cláusula `FROM`.

### Isolar Trechos Independentes (`UNION` para `OR`)

O uso de `OR` em critérios de filtro que envolvem múltiplas junções pode ser problemático porque:
1.  **Impede Índices:** A presença do `OR` pode impedir o uso de índices nos filtros.
2.  **Gera Tabela Intermediária Grande:** A consulta pode gerar uma tabela intermediária enorme antes de aplicar os filtros.

*   **Otimização:** Se os filtros são independentes, a solução é **isolar cada filtro em um `SELECT` separado** e combiná-los usando `UNION`. Isso permite que os índices sejam usados em cada subconsulta e reduz o volume de dados intermediário.

**Exemplo de Otimização (Substituindo `OR` por `UNION`):**
```sql
-- Versão Lenta (Junções desnecessárias e OR)
SELECT DISTINCT nomeP FROM proj LEFT JOIN aloc ... LEFT JOIN ativ ...
WHERE aloc.horas > 20 OR ativ.horas > 20;

-- Versão Otimizada (Isolando a lógica com UNION)
(SELECT nomeP FROM proj NATURAL JOIN aloc WHERE aloc.horas > 20)
UNION
(SELECT nomeP FROM proj NATURAL JOIN ativ WHERE ativ.horas > 20);
```

***

## 4. Junções Avançadas: Semi-Join e Anti-Join

Esses tipos de junção, expressos via subconsultas, são mais eficientes do que simular a mesma lógica com junções externas ou junções regulares seguidas de `DISTINCT`.

### Semi-junção

*   **Definição:** Retorna tuplas da **parte principal** que possuem **pelo menos uma correspondência** na parte secundária. Cada tupla é retornada apenas uma vez.
*   **Sintaxe SQL:** Usa `IN` (subconsulta independente) ou `EXISTS` (subconsulta correlacionada).
    ```sql
    -- Usando EXISTS (Parte principal: movie; Parte secundária: movie_cast)
    SELECT title FROM movie m WHERE EXISTS (SELECT 1 FROM movie_cast mc WHERE m.idM = mc.idM );
    -- Usando IN (Subconsulta retorna a lista de IDs de filme)
    SELECT title FROM movie WHERE idM IN (SELECT idM FROM movie_cast);
    ```
*   **Representação Interna:** O SGBD usa **`Nested Loop Left Semi Join`** ou variações do **`Hash Semi Join`**. O algoritmo **apenas verifica se a correspondência existe** na parte secundária, sem realizar o cruzamento completo dos dados.
*   **Eficiência vs. Junção Regular:** O Semi-Join é preferível a `INNER JOIN` + `DISTINCT` porque evita o cruzamento sobressalente e o *overhead* de remoção de duplicatas (`Hash Duplicate Removal`).

### Anti-junção

*   **Definição:** Retorna tuplas da **parte principal** que **não possuem correspondência** na parte secundária. Cada tupla é retornada apenas uma vez.
*   **Sintaxe SQL:** Usa `NOT EXISTS` ou `NOT IN`.
    ```sql
    -- Filmes que não têm membros do elenco registrados
    SELECT title FROM movie m WHERE NOT EXISTS (SELECT 1 FROM movie_cast mc WHERE m.idM = mc.idM );
    ```
*   **Representação Interna:** O SGBD usa **`Nested Loop Left Anti Join`** ou **`Hash Left Anti Join/Right Anti Join`**. O algoritmo apenas verifica a **não existência** da correspondência.
*   **Eficiência vs. Junção Externa:** O Anti-Join é mais eficiente do que simular com `LEFT JOIN` + `WHERE mc.idM IS NULL`, pois evita o trabalho de localizar os cruzamentos e depois descartá-los.

### `NOT IN` vs. `NOT EXISTS`

O uso de `NOT IN` deve ser evitado se a subconsulta puder retornar valores **nulos**.

*   **Comportamento do `NOT IN` com `NULL`:** Qualquer comparação com nulo é tratada como **incerta** (UNKNOWN) pelo SGBD, e informações incertas **nunca são retornadas**. Se a subconsulta retornar um único `NULL`, a consulta `NOT IN` retornará um conjunto vazio, mesmo que haja registros que deveriam ser retornados.
*   **Comportamento do `NOT EXISTS`:** O `NOT EXISTS` não é afetado por nulos na subconsulta, dando mais liberdade ao otimizador.

***

## 5. Operadores SQL Avançados

### Common Table Expression (CTE)

A CTE é uma expressão de tabela temporária definida pela cláusula `WITH`.

*   **Objetivo:** Facilita a leitura, manutenção e reutilização de subconsultas complexas. É uma alternativa temporária às `VIEWs`.
*   **Sintaxe Básica:**
    ```sql
    WITH nome_da_cte AS (
        SELECT AVG(custo) AS media FROM proj -- subconsulta
    )
    SELECT p.*
    FROM proj p
    JOIN nome_da_cte m ON p.custo > m.media; -- usa a CTE como uma tabela
    ```
*   **Recursividade:** CTEs podem ser usadas de forma recursiva (com `WITH RECURSIVE`) para processar dados hierárquicos, como a subordinação de funcionários.

**Exemplo de CTE Recursiva (Funcionário 1):**
```sql
WITH RECURSIVE hierarquia AS (
    -- Parte Base: começa com o idFunc = 1
    SELECT idFunc, idChefe, 1 AS nivel FROM func WHERE  idFunc = 1
    UNION ALL
    -- Parte recursiva: pega quem tem chefe na hierarquia já descoberta
    SELECT f.idFunc, f.idChefe, h.nivel + 1
    FROM func f
    INNER JOIN hierarquia h ON f.idChefe = h.idFunc
)
SELECT * FROM hierarquia;
```

### Funções de Janela (`Window Functions`)

As Funções de Janela permitem executar cálculos sobre um conjunto de linhas relacionadas à linha atual, **sem agrupar os dados** (diferente do `GROUP BY`).

*   **Sintaxe:** A cláusula `OVER` define a janela.
    ```sql
    funcao() OVER (
        PARTITION BY ...  -- Divide os dados em grupos lógicos (janelas)
        ORDER BY ...      -- Define a ordem dentro da janela
    )
    ```
*   **Funções de Ranqueamento:**
    *   `ROW_NUMBER()`: Numera as linhas sequencialmente.
    *   `RANK()`: Dá um ranking, empata valores e **deixa buracos**.
    *   `DENSE_RANK()`: Dá um ranking, empata valores, mas **não deixa buracos** na sequência.
*   **Funções Agregadas na Janela:**
    *   `AVG()` ou `SUM()`: Sem `ORDER BY` na janela, calcula o agregado sobre toda a partição e repete o valor em cada linha.
    *   `SUM()` Acumulada: Com `ORDER BY` na cláusula `OVER`, a soma é calculada cumulativamente.

**Exemplo de Ranqueamento (`DENSE_RANK`):**
```sql
SELECT status, nome, custo,
DENSE_RANK() OVER (PARTITION BY status ORDER BY custo ) AS ordem
FROM Projeto;
```

***

## 6. Estatísticas para Estimativa de Custo

O otimizador utiliza estatísticas para estimar quantos registros serão retornados, auxiliando na escolha dos melhores algoritmos (custos) e ordem das operações.

### Estatísticas Principais

*   $\mathbf{n(r)}$: Número de tuplas (registros) na relação $r$.
*   $\mathbf{v(A, r)}$: Cardinalidade do atributo $A$, ou seja, o **número de valores distintos** para o atributo $A$.
*   **Seletividade (P):** Probabilidade de um filtro ser verdadeiro. Quanto mais alta a probabilidade, **menos seletivo** é o filtro.

### Estimativa com Filtros

1.  **Igualdade (`WHERE tab.A = valor`):** A seletividade é estimada como o inverso da cardinalidade.
    $$\mathbf{P(A) = \frac{1}{v(A, tab)}}$$
    $$\mathbf{\text{Estimativa} = n(tab) \cdot P(A)}$$
    *Exemplo:* Em `proj`, $n=2.000$ e $v(\text{custo})=50$. $\text{Estimativa} = 2.000 \cdot (1/50) = \mathbf{40}$ registros.

2.  **Conjunção (`AND`):** Para eventos independentes, a probabilidade da interseção é o produto das probabilidades individuais.
    $$\mathbf{P(A \cap B) = P(A) \cdot P(B)}$$
    *Exemplo:* $P(A) = 0.1$ e $P(B) = 0.5$. $P(A \cap B) = 0.05$. $\text{Estimativa} = 2.000 \cdot 0.05 = \mathbf{100}$.

3.  **Disjunção (`OR`):**
    $$\mathbf{P(A \cup B) = P(A) + P(B) - P(A \cap B)}$$
    Se os filtros são mutuamente exclusivos (ex: `custo = 100k OR custo = 200k`), $P(A \cap B)$ é zero, simplificando para a soma das probabilidades.

### Estimativa com Junções (PK/FK)

Em junções por chave primária (PK) e chave estrangeira (FK), a estimativa é baseada na tabela FK e na seletividade dos filtros aplicados na tabela PK.

$$\mathbf{\text{Estimativa} = n(FK) \cdot \text{sel}(PK)}$$

O **máximo de registros** gerados é equivalente ao tamanho da tabela que contém a FK.

*Exemplo:* Junção `proj` (PK) e `aloc` (FK). $n(\text{aloc}) = 100.000$. Filtro na PK: `custo=30.000`. $v(\text{custo}, \text{proj}) = 10$.
1.  $\text{sel}(\text{PK}) = 1/10$.
2.  $\text{Estimativa} = 100.000 \cdot 1/10 = \mathbf{10.000}$ registros.

***

## 7. Controle de Concorrência e Transações

### Transações e Propriedades ACID

Uma **transação** é uma unidade de trabalho lógica que consiste em operações de `READ` e `WRITE` que acessam e possivelmente alteram registros.

*   **Sintaxe em SQL:** A transação é delimitada por `START TRANSACTION` e finalizada por `COMMIT` (sucesso) ou `ROLLBACK` (falha/aborto).
*   **Estados:** Uma transação passa pelos estados: **Active** (executando) $\rightarrow$ **Partially committed** (último comando executado) $\rightarrow$ **Committed** (sucesso e durável) ou **Failed** $\rightarrow$ **Aborted** (revertida).

As propriedades **ACID** garantem a integridade do banco:

| Propriedade | Objetivo | Requisito de Integridade |
| :--- | :--- | :--- |
| **Atomicidade** | **Tudo ou nada.** Ou todas as operações são refletidas, ou nenhuma é. O `ROLLBACK` é usado para desfazê-la. |
| **Consistência** | Levar o banco de um estado correto para outro, preservando as regras de integridade (ex: valor transferido = valor debitado). |
| **Isolamento** | Fazer com que transações concorrentes pareçam ser executadas **serialmente**, evitando que uma veja dados inconsistentes de outra. |
| **Durabilidade** | Garantir que, após o `COMMIT`, as atualizações persistam permanentemente, mesmo em caso de falhas. |

### Schedules e Concorrência

Um *schedule* é a ordem cronológica em que as instruções de transações concorrentes são executadas. O objetivo dos protocolos de concorrência é gerar schedules **concorrentes e consistentes**.

*   **Schedules Concorrentes:** Instruções são executadas de forma intercalada.
*   **Schedules Seriais:** Transações são executadas uma após a outra, sem intercalação.

Um protocolo de concorrência deve gerar schedules que sejam:

1.  **Recuperáveis:** Se uma transação ($T_A$) lê um item escrito por outra ($T_B$), o **commit de $T_B$ deve aparecer antes do commit de $T_A$**. Isso é imprescindível para a consistência.
2.  **Sem Rollback em Cascata:** Se $T_A$ lê um item escrito por $T_B$, o **commit de $T_B$ deve ocorrer antes que $T_A$ tenha feito a leitura**. Isso previne que o abortar de uma transação leve ao abortar de várias outras que leram seus dados. **Todo schedule sem cascata é recuperável**.

### Protocolos Baseados em Lock

Um *lock* é um mecanismo que controla o acesso concorrente a um item de dados.

*   **Lock Compartilhado (S):** Permite apenas **leitura**. Várias transações podem ter locks S sobre o mesmo item.
*   **Lock Exclusivo (X):** Permite **leitura e escrita**. Se uma transação tem um lock X, nenhuma outra transação pode obter qualquer tipo de lock sobre o item.
*   **Matriz de Compatibilidade:** Locks S são compatíveis entre si ("Sim"). Locks X não são compatíveis com S ou X.

#### Protocolo Two-Phase Locking (2PL)

O 2PL exige que as transações passem por duas fases:

1.  **Fase de Crescimento:** A transação pode **obter locks**, mas não pode liberá-los.
2.  **Fase de Encolhimento:** A transação pode **liberar locks**, mas não pode obter novos locks.

**Limitações do 2PL Puro:** O 2PL puro **não previne rollbacks em cascata** e **gera schedules não recuperáveis**.

#### Protocolo Rigorous Two-Phase Locking

Esta é uma variação mais segura do 2PL.

*   **Regra:** Uma transação deve **segurar todos os locks** (S ou X) **até a instrução final de `COMMIT` ou `ABORT`**.
*   **Vantagens:** O protocolo rigoroso **previne rollbacks em cascata** e **gera somente schedules recuperáveis**. O bloqueio e desbloqueio são realizados automaticamente.

### Protocolos Multiversionados (MVCC)

São protocolos **menos bloqueantes** do que os baseados em *lock*, muito usados na prática.

*   **Característica:** Baseiam-se em **múltiplas versões** de um registro.
*   **Benefício Principal:** Leitura **não bloqueia** escrita, e escrita **não bloqueia** leitura.

### `SELECT FOR SHARE`/`SELECT FOR UPDATE`

O comando `SELECT ... FOR UPDATE` é usado para obter um lock exclusivo sobre os registros lidos em uma transação, garantindo o isolamento durante operações como transferências bancárias.

```sql
START TRANSACTION;
SELECT saldo INTO s1 FROM conta WHERE id = 1 FOR UPDATE; -- Obtém lock X sobre conta 1
-- ... lógica segura ...
UPDATE conta SET saldo = saldo + 50 WHERE id = 1;
COMMIT;
```
