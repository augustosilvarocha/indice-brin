# Índice Brin
Projeto acadêmico de Banco de Dados — Análise prática do uso de índices BRIN em tabelas massivas de sensores IoT utilizando PostgreSQL.

## Objetivo

Demonstrar, de forma prática e comparativa, o funcionamento do índice **BRIN (Block Range Index)** em cenários com bilhões de registros, explicando:

- O que são índices no PostgreSQL
- Diferenças entre **BRIN** e **B-Tree**
- Casos em que o BRIN é mais vantajoso
- Ganhos em desempenho e economia de espaço em disco

## Estrutura do Projeto

- Criação da tabela de sensores
- Inserção de dados com `generate_series`
- Criação e remoção dos índices B-Tree e BRIN
- Consultas com e sem índice, usando `EXPLAIN ANALYZE`
- Prints das execuções para comparação de desempenho
- Tabela com os tamanhos dos índices em diferentes volumes

### O que é um índice?

Um **índice** é uma estrutura auxiliar que acelera a busca por dados em tabelas. Funciona como o índice de um livro: ao invés de ler todas as páginas (linhas da tabela), você vai direto ao ponto.

### Índice B-Tree

- Organiza os dados de forma ordenada
- Ideal para buscas específicas ou intervalos pequenos
- Mais preciso, mas consome muito espaço

### Índice BRIN

- Divide a tabela em **blocos de páginas** (por exemplo, de 128KB)
- Armazena o **mínimo e máximo** de cada bloco para a coluna indexada
- Ideal para dados **ordenados fisicamente** (ex: logs, datas)
- Extremamente leve

---

## Demonstração Prática

### Criação da Tabela

```sql
CREATE TABLE sensores (
  id SERIAL PRIMARY KEY,
  data_hora_evento TIMESTAMP,
  tipo TEXT,
  valor DOUBLE PRECISION
);
```
### Resumo dos campos:
- **id**: identificador único de cada registro.
- **data_hora_evento**: registra quando o dado foi coletado.
- **tipo**: tipo do sensor (temperatura, umidade, etc).
- **valor**: valor da leitura do sensor.

### Inserindo os dados
```sql
INSERT INTO sensores (data_hora_evento, tipo, valor)
SELECT 
    NOW() + (s * INTERVAL '1 second'),
    CASE 
        WHEN s % 3 = 0 THEN 'temperatura'
        WHEN s % 3 = 1 THEN 'umidade'
        ELSE 'pressao'
    END,
    CASE 
        WHEN s % 3 = 0 THEN 20 + random() * 15
        WHEN s % 3 = 1 THEN 40 + random() * 60
        ELSE 980 + random() * 40
    END
FROM generate_series(1, 200000000) AS s;
```
Este script insere 20 milhões de registros simulados, alternando entre sensores de temperatura, umidade e pressão.
Você pode ajustar a quantidade modificando o valor final em generate_series(1, 20000000).

## Consultas com EXPLAIN ANALYZE

### Sem índice (Sequential Scan)

- Leitura completa da tabela  
- Muito lento para grandes volumes

Sem índice, o banco de dados realiza uma varredura sequencial, o que compromete o desempenho das consultas.  
A criação de um índice B-Tree melhora consideravelmente a velocidade ao buscar por colunas indexadas, como `data_hora_evento`.

### Criar e Medir Índice B-Tree
```sql
CREATE INDEX idx_btree ON sensores (data_hora_evento);
SELECT pg_size_pretty(pg_relation_size('idx_btree'));
```
Após a criação do índice, o segundo comando retorna o espaço ocupado por ele no disco.
Com 20 milhões de registros, o índice B-Tree gerado ocupou aproximadamente 4284 MB (ou 4,284 GB).

### Consulta com Índice B-Tree

```sql
EXPLAIN ANALYZE
SELECT * FROM sensores
WHERE data_hora_evento BETWEEN '2026-03-02 00:00:00' AND '2026-10-30 23:59:59';
```
🕒 Tempo de execução: entre 15 a 20 segundos.

### Consulta com Índice Brin
```sql
EXPLAIN ANALYZE
SELECT * FROM sensores
WHERE data_hora_evento BETWEEN '2026-03-02 00:00:00' AND '2026-10-30 23:59:59';
```
🕒 Tempo de execução: entre 10 a 12 segundos.

## Análise Comparativa
O resultado do `EXPLAIN ANALYZE` fornece dados importantes para comparar o desempenho dos índices BRIN e B-Tree em consultas sobre grandes volumes de dados.
Para facilitar a compreensão, criei a tabela abaixo que resume a performance de cada índice.

| Critério                 | **BRIN**         | **B-Tree**      | Melhor     |
| ------------------------ | ---------------- | --------------- | ---------- |
| **Tipo de leitura**      | Bitmap Heap Scan | Index Scan      | Depende    |
| **Blocos lidos (lossy)** | 161.536 blocos   | Linhas precisas | BRIN       |
| **Linhas retornadas**    | 20.995.199       | 20.995.199      | Igual      |
| **Planejamento**         | 0.121 ms         | 0.345 ms        | BRIN       |
| **Tempo total**          | 10.2 segundos    | 15.8 segundos   | BRIN       |

Em algumas situações, consultas específicas podem se beneficiar mais do índice B-Tree, principalmente quando a precisão na leitura de dados é prioritária.  
O índice BRIN, por sua vez, é uma estrutura leve que funciona muito bem para tabelas gigantes com dados ordenados fisicamente, sendo bastante usado em cenários como logs de sensores e dados temporais com inserção sequencial.

## Conclusão

- O índice **BRIN** não substitui o **B-Tree** em todos os casos, mas é excelente para tabelas muito grandes com colunas ordenadas.
- É ideal para logs de sensores, dados temporais e cenários de inserção sequencial.
- Proporciona um ganho significativo de espaço, reduzindo o tamanho do índice de centenas de MB para poucos KB.
- Melhora consideravelmente o tempo de resposta em consultas por intervalo de dados.

### Quando usar BRIN?

- Quando os dados possuem ordem física natural (ex: por data ou ID sequencial)
- Para consultas que filtram por intervalos grandes
- Em situações que exigem economia de espaço no armazenamento de índices

