# Guia de Referência — SQL para Banco Relacional (MySQL)

Documentação direta ao ponto: o que cada comando faz, quando usar e exemplo pronto.

---

## 1. DDL — Data Definition Language

Comandos que definem a estrutura do banco: criação e alteração de bancos, tabelas, tipos de dados e restrições (constraints).

### Comandos de banco e tabela

```sql
CREATE DATABASE IF NOT EXISTS aula;
USE aula;
SELECT DATABASE();           -- mostra o banco atual
SHOW TABLES;                 -- lista tabelas do banco
DESCRIBE cliente;            -- mostra estrutura da tabela
DROP TABLE IF EXISTS tabela; -- remove tabela (irreversível)
```

### Tipos de dados mais usados

| Tipo | O que é / quando usar |
|---|---|
| `INT` | Número inteiro. IDs, contadores, quantidades. |
| `DECIMAL(10,2)` | Número decimal exato — 10 dígitos no total, 2 depois da vírgula. Ideal para valores monetários (nunca use FLOAT para dinheiro). |
| `VARCHAR(n)` | Texto de tamanho variável até n caracteres. Nomes, e-mails, descrições. |
| `CHAR(n)` | Texto de tamanho fixo. Siglas (UF), códigos (CEP), valores sempre do mesmo tamanho. |
| `DATE` / `DATETIME` | DATE = só data (AAAA-MM-DD). DATETIME = data + hora. |
| `BOOLEAN` | Verdadeiro/Falso (internamente TINYINT 0/1). Flags como "ativo". |
| `TEXT` | Texto longo sem limite prático. Descrições extensas, observações. |

### Constraints (restrições)

| Constraint | O que garante |
|---|---|
| `PRIMARY KEY` | Identificador único da linha. Não repete, não pode ser NULL. |
| `AUTO_INCREMENT` | Gera o valor automaticamente, incrementando a cada novo registro. |
| `FOREIGN KEY` | Garante que o valor exista em outra tabela (integridade referencial). |
| `NOT NULL` | Obriga o preenchimento do campo. |
| `UNIQUE` | Não permite valores duplicados (ex: e-mail, CPF). |
| `DEFAULT` | Valor usado automaticamente se nada for informado no INSERT. |
| `CHECK` | Valida uma condição antes de aceitar o dado (ex: `genero IN ('M','F','O')`). |

### Exemplo completo de tabela

```sql
CREATE TABLE cliente (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nome VARCHAR(100) NOT NULL,
    genero CHAR(1) CHECK (genero IN ('M', 'F', 'O')),
    dataNascimento DATE,
    salario DECIMAL(10,2) UNSIGNED DEFAULT 0,
    email VARCHAR(120) UNIQUE,
    cidadeId INT NOT NULL,
    ativo BOOLEAN DEFAULT TRUE,
    dataCadastro DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (cidadeId) REFERENCES cidade(id)
);
```

> ⚡ **Dica:** `UNSIGNED` impede números negativos — útil para salário, preço, estoque, idade.

---

## 2. DML — Data Manipulation Language

Comandos que manipulam os dados dentro das tabelas: inserir, atualizar e excluir registros.

### INSERT — inserir registros

```sql
-- Um registro
INSERT INTO estado (id, nome, sigla) VALUES (1, 'Paraná', 'PR');

-- Vários registros de uma vez (mais eficiente)
INSERT INTO cidade (id, nome, estadoId) VALUES
(1, 'Curitiba', 1),
(2, 'São Paulo', 2),
(3, 'Bagé', 3);
```

### UPDATE — atualizar registros

```sql
-- Update simples
UPDATE cliente SET salario = salario * 1.10 WHERE nome LIKE '%Helena%';

-- Update com subconsulta (valor vem de outra tabela)
UPDATE funcionario
SET cidadeId = (SELECT id FROM cidade WHERE nome = 'Curitiba')
WHERE matricula = 2;
```

> ⚠ **Atenção:** Sempre use `WHERE` no UPDATE. Sem WHERE, o comando altera TODAS as linhas da tabela.

### DELETE — excluir registros

```sql
-- Delete pontual
DELETE FROM cliente WHERE id = 5;

-- Delete em lote com subconsulta
DELETE FROM cliente WHERE cidadeId IN (SELECT id FROM cidade WHERE nome = 'Bagé');
```

> ⚠ **Atenção:** Assim como o UPDATE, DELETE sem WHERE apaga toda a tabela. Em produção, prefira rodar dentro de uma transação (ver seção 9) para poder reverter (ROLLBACK) se algo der errado.

---

## 3. DQL — Data Query Language (SELECT)

O comando mais usado no dia a dia: consultar, filtrar e ordenar dados.

### Estrutura básica

```sql
SELECT colunas
FROM tabela
WHERE condição
GROUP BY coluna
HAVING condição_do_grupo
ORDER BY coluna
LIMIT n;
```

> ⚡ **Dica:** Essa é a ordem que o SQL processa internamente (não a ordem que você escreve): `FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT`. Por isso o HAVING filtra grupos e o WHERE filtra linhas individuais.

### Seleção, alias e ordenação

```sql
-- Colunas com alias (nome amigável no resultado)
SELECT
    nome AS 'Nome do Cliente',
    salario AS 'Salário Atual',
    salario * 1.10 AS 'Salário Reajustado'
FROM cliente;

-- Ordenação: ASC (padrão) ou DESC
SELECT * FROM funcionario
ORDER BY salario DESC, nome ASC
LIMIT 5;
```

### Operadores de filtro (WHERE)

| Operador | O que faz |
|---|---|
| `=, <>, >, <, >=, <=` | Comparações básicas. |
| `AND / OR` | Combina condições. Atenção: AND é avaliado antes do OR — use parênteses para deixar explícito. |
| `LIKE '%texto%'` | Busca por padrão de texto. `%` = qualquer sequência, `_` = um caractere. |
| `IN (lista)` | Verifica se o valor está dentro de uma lista de opções. |
| `BETWEEN a AND b` | Verifica se o valor está dentro de um intervalo (inclusive). |
| `IS NULL / IS NOT NULL` | Verifica se o campo está vazio. Nunca use `= NULL` (não funciona em SQL). |

```sql
SELECT * FROM cliente
WHERE nome LIKE '%Silva%'
   OR (cidadeId IN (1, 2, 3)
   AND salario BETWEEN 7000 AND 10000);
```

> ⚡ **Dica:** Coloquei parênteses no exemplo acima porque sem eles o AND seria avaliado primeiro, mudando o resultado da busca.

### CASE — lógica condicional dentro do SELECT

```sql
SELECT
    nome,
    CASE
        WHEN genero = 'M' THEN 'Masculino'
        WHEN genero = 'F' THEN 'Feminino'
        ELSE 'Outro'
    END AS genero_desc
FROM cliente;
```

---

## 4. Funções Nativas (numéricas, texto e data)

Funções prontas do MySQL para transformar e calcular valores diretamente na consulta.

### Numéricas

| Função | O que faz |
|---|---|
| `ROUND(n, d)` | Arredonda n para d casas decimais. |
| `TRUNCATE(n, d)` | Corta as casas decimais sem arredondar. |
| `MOD(a, b)` | Resto da divisão de a por b. |
| `ABS(n)` | Valor absoluto (remove sinal negativo). |

```sql
SELECT
    salario,
    ROUND(salario, 2) AS arredondado,
    TRUNCATE(salario, 1) AS truncado,
    MOD(matricula, 2) AS resto
FROM funcionario;
```

### Texto (strings)

| Função | O que faz |
|---|---|
| `UPPER / LOWER` | Converte para maiúsculas / minúsculas. |
| `LENGTH(texto)` | Quantidade de caracteres. |
| `SUBSTRING(texto, ini, qtd)` | Extrai parte do texto a partir da posição "ini". |
| `REPLACE(texto, de, para)` | Substitui um trecho por outro dentro do texto. |
| `TRIM(texto)` | Remove espaços em branco do início e do fim. |
| `CONCAT(a, b, ...)` | Junta vários valores/textos em um só. |

```sql
SELECT
    nome,
    UPPER(nome) AS maiusculo,
    LENGTH(nome) AS tamanho,
    SUBSTRING(nome, 1, 5) AS inicial,
    REPLACE(email, '@', '[at]') AS email_mod
FROM cliente;
```

### Data e hora

| Função | O que faz |
|---|---|
| `CURDATE() / NOW()` | Data atual / data e hora atual. |
| `DATE_ADD(data, INTERVAL n unidade)` | Soma um período à data (DAY, MONTH, YEAR...). |
| `DATEDIFF(d1, d2)` | Diferença em dias entre duas datas. |
| `DAYNAME(data)` | Nome do dia da semana. |

```sql
SELECT
    CURDATE() AS hoje,
    NOW() AS agora,
    DATE_ADD(CURDATE(), INTERVAL 30 DAY) AS mais_30,
    DATEDIFF('2026-12-31', CURDATE()) AS dias_ate_ano_novo,
    DAYNAME(CURDATE()) AS dia_semana;
```

---

## 5. Funções de Agregação e GROUP BY

Resumem várias linhas em um único valor — totais, médias, contagens — opcionalmente agrupados por categoria.

### Funções de agregação

| Função | O que faz |
|---|---|
| `COUNT(*)` | Conta o total de linhas. `COUNT(coluna)` ignora valores NULL. |
| `SUM(coluna)` | Soma os valores da coluna. |
| `AVG(coluna)` | Calcula a média. |
| `MIN / MAX(coluna)` | Menor / maior valor. |

```sql
SELECT
    COUNT(*) AS total_clientes,
    COUNT(genero) AS com_genero,
    AVG(salario) AS media_salario,
    MIN(salario) AS menor_salario,
    MAX(salario) AS maior_salario,
    SUM(salario) AS total_folha
FROM cliente;
```

### GROUP BY + HAVING

```sql
SELECT
    departamento,
    COUNT(*) AS qtd,
    AVG(salario) AS media
FROM funcionario
GROUP BY departamento
HAVING COUNT(*) > 1
ORDER BY media DESC;
```

> ⚡ **Regra de ouro:** toda coluna no SELECT que não está dentro de uma função de agregação precisa estar no GROUP BY. Use HAVING para filtrar grupos (ex: "departamentos com mais de 1 funcionário") — WHERE não funciona aqui porque ainda não existe grupo formado nessa etapa.

---

## 6. JOINs — Combinando Tabelas

Permitem buscar dados relacionados que estão espalhados em tabelas diferentes, através das chaves estrangeiras.

### Tipos de JOIN

| Tipo | O que faz |
|---|---|
| `INNER JOIN` | Retorna apenas linhas que têm correspondência nas DUAS tabelas. |
| `LEFT JOIN` | Retorna todas as linhas da tabela da esquerda, mesmo sem correspondência (campos da direita ficam NULL). |
| `RIGHT JOIN` | O mesmo que LEFT JOIN, mas priorizando a tabela da direita. |
| `SELF JOIN` | A tabela se junta com ela mesma — útil para hierarquias (funcionário/gerente). |

### INNER JOIN — só o que casa nas duas tabelas

```sql
SELECT
    f.nome AS funcionario,
    c.nome AS cidade,
    e.sigla AS estado
FROM funcionario f
INNER JOIN cidade c ON f.cidadeId = c.id
INNER JOIN estado e ON c.estadoId = e.id;
```

### LEFT JOIN — tudo da tabela principal, mesmo sem par

```sql
SELECT
    c.nome AS cidade,
    e.nome AS estado
FROM cidade c
LEFT JOIN estado e ON c.estadoId = e.id;
```

### Encontrar registros "órfãos" (sem correspondência)

```sql
-- Cidades que NÃO têm estado correspondente
SELECT c.nome AS cidade
FROM cidade c
LEFT JOIN estado e ON c.estadoId = e.id
WHERE e.id IS NULL;
```

> ⚡ **Dica:** Esse é o padrão clássico para achar "o que está faltando": `LEFT JOIN` + `WHERE tabela_da_direita.id IS NULL`.

### SELF JOIN — hierarquia (funcionário/gerente)

```sql
SELECT
    f.nome AS funcionario,
    g.nome AS gerente
FROM funcionario f
LEFT JOIN funcionario g ON f.gerente = g.matricula;
```

> ⚡ **Dica:** LEFT JOIN aqui é importante porque funcionários sem gerente (ex: o diretor) ainda aparecem na lista, com "gerente" = NULL.

---

## 7. Subconsultas, Views e Índices

Recursos para reaproveitar lógica de consulta e melhorar performance.

### Subconsultas (subqueries)

```sql
-- Subconsulta simples: resultado fixo usado como filtro
SELECT nome, salario
FROM funcionario
WHERE salario = (SELECT MAX(salario) FROM funcionario);

-- Com IN: filtra por uma lista vinda de outra tabela
SELECT nome
FROM cliente
WHERE cidadeId IN (SELECT id FROM cidade WHERE nome = 'Curitiba');

-- Correlacionada: a subconsulta depende de cada linha da consulta externa
SELECT nome, salario
FROM funcionario f
WHERE salario > (SELECT AVG(salario) FROM funcionario WHERE departamento = f.departamento);
```

> ⚡ **Dica:** Subconsulta correlacionada roda uma vez para cada linha da consulta externa — útil, mas pode ficar lenta em tabelas grandes. Em casos assim, considere reescrever com JOIN.

### VIEW — consulta salva como "tabela virtual"

```sql
CREATE OR REPLACE VIEW vw_funcionarios AS
SELECT
    matricula,
    nome,
    salario,
    departamento,
    (SELECT nome FROM cidade WHERE id = f.cidadeId) AS cidade
FROM funcionario f;

SELECT * FROM vw_funcionarios;
```

> ⚡ **Dica:** Use VIEW quando uma consulta complexa é reutilizada com frequência (relatórios, dashboards) ou quando quer simplificar o acesso para outros usuários sem expor a estrutura real das tabelas.

### Índices — acelerar buscas

```sql
CREATE INDEX idx_cliente_nome ON cliente(nome);
CREATE INDEX idx_funcionario_salario ON funcionario(salario);

SHOW INDEX FROM cliente;
```

> ⚡ **Dica:** Crie índices em colunas usadas com frequência em WHERE, JOIN e ORDER BY. Evite criar índices em excesso: cada índice acelera a leitura, mas deixa INSERT/UPDATE/DELETE mais lentos.

---

## 8. Triggers, Functions e Stored Procedures

Lógica que vive dentro do banco: executa automaticamente (trigger) ou é chamada sob demanda (function/procedure).

### Diferenças rápidas

| Recurso | O que é |
|---|---|
| `TRIGGER` | Executa automaticamente quando ocorre um evento (INSERT, UPDATE, DELETE) em uma tabela. Não é chamado manualmente. |
| `FUNCTION` | Recebe parâmetros e retorna um único valor. Pode ser usada dentro de um SELECT, como qualquer função nativa. |
| `PROCEDURE` | Bloco de lógica chamado explicitamente com CALL. Pode ter parâmetros de entrada (IN), saída (OUT) ou ambos (INOUT), e não retorna valor direto como a função. |

### TRIGGER — exemplo de auditoria

```sql
DELIMITER $$
CREATE TRIGGER trg_auditoria_cliente
AFTER UPDATE ON cliente
FOR EACH ROW
BEGIN
    INSERT INTO auditoria (acao, tabela, registroId, dadosAntigos, dadosNovos)
    VALUES ('UPDATE', 'cliente', OLD.id,
            CONCAT('Salario: ', OLD.salario),
            CONCAT('Salario: ', NEW.salario));
END$$
DELIMITER ;
```

> ⚡ **Dica:** `OLD` = valor antes da alteração, `NEW` = valor depois. `AFTER` = executa depois do comando concluído; `BEFORE` = executa antes (útil para validar/ajustar dados antes de salvar).

### FUNCTION — exemplo de cálculo reutilizável

```sql
DELIMITER $$
CREATE FUNCTION reajuste_salario(sal DECIMAL(10,2), perc DECIMAL(5,2))
RETURNS DECIMAL(10,2)
DETERMINISTIC
BEGIN
    RETURN sal * (1 + perc/100);
END$$
DELIMITER ;

SELECT nome, salario, reajuste_salario(salario, 10) AS novo_salario
FROM funcionario;
```

> ⚡ **Dica:** `DETERMINISTIC` indica que, para os mesmos parâmetros de entrada, o resultado é sempre o mesmo — o MySQL exige essa declaração para certas funções.

### PROCEDURE — exemplo com validação e saída

```sql
DELIMITER $$
CREATE PROCEDURE sp_inclui_cliente(
    IN p_nome VARCHAR(100),
    IN p_genero CHAR(1),
    IN p_salario DECIMAL(10,2),
    IN p_cidadeId INT,
    OUT p_mensagem VARCHAR(100)
)
BEGIN
    IF p_salario < 0 THEN
        SET p_mensagem = 'Salário inválido';
    ELSEIF LENGTH(TRIM(p_nome)) = 0 THEN
        SET p_mensagem = 'Nome obrigatório';
    ELSE
        INSERT INTO cliente (nome, genero, salario, cidadeId)
        VALUES (p_nome, p_genero, p_salario, p_cidadeId);
        SET p_mensagem = 'Cliente incluído com sucesso';
    END IF;
END$$
DELIMITER ;

-- Chamada
CALL sp_inclui_cliente('Teste', 'M', 5000, 1, @msg);
SELECT @msg;
```

---

## 9. Usuários, Permissões e Transações

Controle de acesso ao banco e garantia de que operações críticas sejam tudo-ou-nada.

### Usuários e permissões (GRANT)

```sql
CREATE USER IF NOT EXISTS 'dev'@'localhost' IDENTIFIED BY 'senha123';
GRANT SELECT, INSERT, UPDATE, DELETE ON aula.* TO 'dev'@'localhost';
FLUSH PRIVILEGES;

SHOW GRANTS FOR 'dev'@'localhost';
```

| Comando | O que faz |
|---|---|
| `CREATE USER` | Cria um novo usuário com senha. |
| `GRANT` | Concede permissões específicas (SELECT, INSERT, etc) sobre um banco/tabela. |
| `REVOKE` | Remove permissões previamente concedidas. |
| `FLUSH PRIVILEGES` | Recarrega as permissões para garantir que a alteração tenha efeito imediato. |

> ⚡ **Dica:** Princípio do menor privilégio — dê ao usuário apenas as permissões que ele realmente precisa. Evite usar `GRANT ALL` em ambientes de produção.

### Transações — operações tudo-ou-nada

```sql
START TRANSACTION;

UPDATE conta SET saldo = saldo - 100 WHERE id = 1;
UPDATE conta SET saldo = saldo + 100 WHERE id = 2;

-- Se tudo ocorreu certo:
COMMIT;

-- Se algo deu errado, desfaz tudo desde o START TRANSACTION:
ROLLBACK;
```

> ⚡ **Dica:** Use transações sempre que uma operação de negócio envolver mais de um comando que precisa funcionar em conjunto (ex: transferência bancária — debitar de uma conta e creditar em outra). Se o segundo UPDATE falhar, o ROLLBACK garante que o primeiro também seja desfeito.

---

## Boas práticas gerais

- **Teste em ambiente de desenvolvimento** — nunca rode comandos novos diretamente em produção sem testar antes.
- **Use DESCRIBE para inspecionar** — antes de escrever queries em uma tabela desconhecida, rode `DESCRIBE tabela;` para ver colunas e tipos.
- **Transações em operações críticas** — UPDATE/DELETE em lote ou que envolvam múltiplas tabelas devem rodar dentro de `START TRANSACTION`.
- **Confira o WHERE antes de executar** — rode um SELECT com o mesmo WHERE antes de um UPDATE/DELETE, para confirmar quais linhas serão afetadas.
