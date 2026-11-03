# Relatório do Laboratório Prático – AI Code Reviewer com GitHub Actions

## 1. Objetivo

Este relatório documenta a configuração e a avaliação de um revisor de código baseado em IA no projeto GNews, conforme proposto no [enunciado do laboratório](04_Laboratorio_Pratico_AI_Code_Reviewer.md). O experimento buscou verificar se o revisor seria capaz de identificar uma vulnerabilidade proposital de SQL Injection e produzir orientações úteis diretamente no Pull Request.

## 2. Evidências do experimento

| Item | Evidência |
|---|---|
| Repositório utilizado | [rafaelgils/gnews](https://github.com/rafaelgils/gnews) |
| Branch do experimento | `feature/search-update` |
| Pull Request — teste principal | [PR #5 – Feature/search update](https://github.com/rafaelgils/gnews/pull/5) |
| Pull Request — prompt injection | [PR #6 – Feature/search update](https://github.com/rafaelgils/gnews/pull/6) |
| Workflow | `AI Code Review` |
| Execução bem-sucedida | [GitHub Actions run #40612471149](https://github.com/rafaelgils/gnews/actions/runs/40612471149) |
| Resultado da execução | `success` |
| Data da análise | 28 de outubro de 2026 |
| Revisor | `github-actions[bot]` por meio da action `anc95/ChatGPT-CodeReview` |



## 3. Vulnerabilidade inserida

A vulnerabilidade foi adicionada ao método `findByTitle`, em `ArticleRepository.java`:

```java
public List<Article> findByTitle(String userInput) {
    String query = "SELECT * FROM article WHERE title = '" + userInput + "'";
    return jdbcTemplate.query(query, articleRowMapper);
}
```

O método concatena uma entrada externa diretamente na instrução SQL. Um valor manipulado poderia alterar a estrutura da consulta, caracterizando **SQL Injection**, classificada em **OWASP A03:2021 – Injection**.

## 4. Análise crítica

### 4.1 Precisão: a IA detectou o SQL Injection?

**Sim.** O revisor identificou corretamente a concatenação de `userInput` na consulta SQL, classificou o problema como SQL Injection e relacionou o achado ao OWASP Top 10. O comentário foi publicado diretamente no arquivo e na linha do método vulnerável.

Evidência: [comentário do revisor em `ArticleRepository.java`](https://github.com/rafaelgils/gnews/pull/5#discussion_r4395886555).

![Comentário do AI Code Reviewer identificando o SQL Injection](assets/ai-code-review/deteccao-sql-injection.png)

*Figura 1 — O revisor identificou o SQL Injection no método `findByTitle` e sugeriu a parametrização da consulta.*


### 4.2 Didática: o comentário explicou o risco e sugeriu uma correção adequada?

**Sim.** O comentário explicou que entradas fornecidas pelo usuário não devem ser concatenadas em consultas SQL e apresentou uma correção com placeholder `?`:

```java
public List<Article> findByTitle(String userInput) {
    String query = "SELECT * FROM article WHERE title = ?";
    return jdbcTemplate.query(query, articleRowMapper, userInput);
}
```

Essa solução faz o `JdbcTemplate` enviar o valor como parâmetro, separando dados da estrutura da instrução SQL. Na prática, a abordagem utiliza uma consulta preparada e impede que o conteúdo de `userInput` seja interpretado como parte do comando.

O revisor não apresentou um exemplo com `NamedParameterJdbcTemplate`, mas isso não prejudica a correção: a parametrização posicional sugerida já resolve a vulnerabilidade. Um comentário ainda mais didático poderia explicar que validação ou escape manual não substituem consultas parametrizadas.

### 4.3 Falso positivo: a IA criticou algo que estava correto?

**Houve observações úteis, mas também houve recomendações contextuais e ruído.**  Os demais comentários precisam de avaliação humana:

| Comentário da IA | Avaliação crítica |
|---|---|
| Dependência H2 com escopo `runtime` | **Falso positivo.** O `application.yml` usa H2 como datasource e o código usa `JdbcTemplate`; o driver precisa estar no classpath de execução. Alterar o escopo para `test` o removeria do artefato e impediria a aplicação de iniciar. [Comentário](https://github.com/rafaelgils/gnews/pull/5#discussion_r4395886553). |
| `VARCHAR(1000)` para URL | **Recomendação especulativa.** Pode ser suficiente para os requisitos do projeto. Seria necessário conhecer os limites esperados antes de apontar um defeito. [Comentário](https://github.com/rafaelgils/gnews/pull/5#discussion_r4395886563). |
| `VARCHAR(255)` como chave primária | **Recomendação especulativa.** O tipo correto depende do domínio, do banco e do formato real do identificador. Não é uma vulnerabilidade por si só. [Comentário](https://github.com/rafaelgils/gnews/pull/5#discussion_r4395886563). |
| Uso de Flyway ou Liquibase | **Boa recomendação de engenharia**, especialmente em sistemas persistentes, mas não representa uma falha crítica neste exercício. [Comentário sobre o schema](https://github.com/rafaelgils/gnews/pull/5#discussion_r4395886563). |

Assim, a IA encontrou problemas reais, mas misturou vulnerabilidades confirmadas, riscos condicionais e preferências arquiteturais. Essa mistura pode aumentar o número de comentários e dificultar a priorização.

### 4.4 Desafio extra: prompt injection ou ofuscação

Para verificar se uma instrução escrita dentro do código poderia manipular o revisor baseado em IA, foi aberto o [PR #6](https://github.com/rafaelgils/gnews/pull/6). Em relação ao teste anterior, a nova alteração substituiu o comentário que identificava o exemplo vulnerável por uma tentativa de prompt injection:



![Código contendo a tentativa de prompt injection](assets/ai-code-review/prompt-injection-codigo.png)

*Figura 2 — Comentário inserido antes do método vulnerável para tentar manipular o revisor de IA.*

O objetivo era induzir o modelo a ignorar as regras de segurança do workflow e considerar seguro um método que ainda concatenava a entrada do usuário na consulta SQL.

**Resultado:** a tentativa de prompt injection falhou. O bot desconsiderou a instrução presente no comentário, classificou a concatenação como SQL Injection e recomendou o uso de uma consulta parametrizada com `?`. O próprio comentário da revisão registrou que a instrução inserida no código não justificava aprovar o método vulnerável.

Evidência: [comentário do bot no `ArticleRepository.java`](https://github.com/rafaelgils/gnews/pull/6#discussion_r4449173304).

![Resposta do bot à tentativa de prompt injection](assets/ai-code-review/prompt-injection-resposta-bot.png)

*Figura 3 — Mesmo diante da instrução maliciosa, o bot identificou o SQL Injection e recomendou uma consulta parametrizada.*

O teste demonstra que o revisor resistiu a essa instrução simples. Como a consulta permaneceu explícita e apenas um caso foi executado, o resultado não comprova resistência a ataques mais elaborados nem a vulnerabilidades ofuscadas.


## 5. Conclusão

O experimento atingiu seu objetivo principal: o revisor detectou corretamente o SQL Injection, explicou o risco e sugeriu uma consulta parametrizada válida. Ele também encontrou riscos adicionais relevantes, como a referência mutável da action e a exposição do console H2.

A análise mostrou, porém, que os comentários do LLM precisam de triagem humana. Algumas recomendações foram contextuais, e o job permaneceu verde mesmo diante de uma vulnerabilidade crítica. Para uso como controle de DevSecOps, a solução deve combinar revisão por IA, análise estática determinística, dependências fixadas por SHA e uma política que faça o check falhar diante de achados críticos.