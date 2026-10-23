# 💸 Budgeting AI — API de orçamento por voz com Spring Boot e Spring AI

Projeto do **Desafio de Projeto DIO – Spring AI**. Parte do projeto base da pasta
[`05-spring-ai`](https://github.com/digitalinnovationone/dio-spring-boot-learning-track/tree/main/05-spring-ai)
e o evolui com **validações de domínio**, uma **nova ferramenta de IA (resumo de gastos)**,
**tratamento de erros HTTP** e **testes automatizados** que rodam sem OpenAI e sem banco.

---

## 📌 O que o projeto faz

Uma API em que a pessoa **fala** o que gastou (ou o que quer saber) e a aplicação responde **por voz**:

```
áudio ──► texto ──► IA escolhe uma ferramenta ──► caso de uso (banco) ──► resposta em texto ──► áudio MP3
        (Whisper)      (ChatClient + @Tool)                                                    (TTS)
```

1. O cliente envia um arquivo de áudio para `POST /transactions/ai`.
2. O áudio é transcrito (`TranscriptionModel`, Whisper).
3. O modelo de linguagem (`ChatClient`) entende a intenção e chama uma **ferramenta** (`@Tool`).
4. A ferramenta executa um **caso de uso real** que grava ou consulta transações no MySQL.
5. A resposta final é convertida em áudio (`TextToSpeechModel`) e devolvida como MP3.

As mesmas regras de negócio são usadas pelos endpoints REST e pela IA: a IA **não** fura as camadas,
ela chama os mesmos casos de uso da camada `application`.

### Endpoints

| Método | Rota | Descrição |
|---|---|---|
| `POST` | `/transactions` | Cria uma transação (JSON) |
| `GET` | `/transactions/{category}` | Lista transações de uma categoria (`GROCERIES`, `PHARMA`, `AUTO`) |
| `GET` | `/transactions/summary` | 🆕 Resumo dos gastos: total geral e total por categoria |
| `POST` | `/transactions/ai` | Recebe áudio (`multipart/form-data`, campo `file`) e devolve a resposta em MP3 |

> **Valores:** são armazenados em **centavos** (`4590` = R$ 45,90) e devolvidos pela API em **reais**.

---

## 🚀 Melhoria implementada

Escolhi uma evolução em quatro frentes pequenas, que se complementam.

### 1. Validações antes de salvar uma transação
A regra mora no **domínio** (`Transaction`), e não no controller, para valer tanto para o REST quanto para a IA:

- descrição obrigatória (não pode ser vazia ou em branco) e com no máximo 255 caracteres;
- valor **maior que zero**;
- categoria obrigatória.

Se alguma regra falhar, é lançada uma `InvalidTransactionException` com mensagem em português.

- **No REST**, um `@RestControllerAdvice` (`ApiExceptionHandler`) converte a exceção em **HTTP 400** com um `ProblemDetail`.
- **No fluxo de voz**, a mensagem do erro volta para o modelo, que explica o problema para a pessoa
  (por exemplo, ao ouvir "gastei zero reais na farmácia"). Esse é o comportamento padrão do Spring AI para erros
  lançados dentro de uma tool; vale conferir na sua versão.

> O construtor usado para **reconstruir** dados já salvos (`toDomain`) não revalida de propósito,
> para que registros antigos no banco nunca quebrem uma leitura.

### 2. Nova ferramenta para o Tool Calling: `summarize-spending`
Novo caso de uso `SummarizeSpendingUseCase`, exposto ao modelo com `@Tool` e também como `GET /transactions/summary`.
Responde perguntas como:

- "Quanto eu gastei no total?"
- "Quanto gastei com farmácia?"
- "Me dá um resumo dos meus gastos."

Todas as categorias aparecem no resumo, inclusive com total zero, para o modelo poder dizer
"você ainda não gastou nada com essa categoria". Os totais são **calculados em Java**, e o prompt manda o modelo
**nunca somar valores por conta própria** (modelos de linguagem erram contas).

Para isso, o contrato `TransactionRepository` ganhou o método `findAll()`, implementado no adaptador JPA.

### 3. Correção de um bug de conversão de valores
O projeto base documentava o valor como **centavos**, mas a saída (`TransactionOutput`) só colocava duas casas
decimais sem dividir por 100, então `1250` centavos apareceria como `1250.00` em vez de `12.50`.
Agora existe `TransactionOutput.toReais(long)`, coberta por teste, e o resumo usa a mesma conversão.

### 4. Prompt do sistema melhorado (`prompts/system-message.st`)
O prompt agora orienta o modelo a:

- converter reais em centavos antes de registrar;
- usar `summarize-spending` para totais e `list-transactions-by-category` para listagens;
- pedir a informação quando faltar valor ou descrição, em vez de inventar;
- responder em frases curtas, sem listas nem símbolos, porque a resposta será **lida em voz alta**.

---

## 🧰 Tecnologias

- Java 25 e Spring Boot 4
- Spring AI 2.0 (`ChatClient`, Tool Calling, `TranscriptionModel`, `TextToSpeechModel`)
- OpenAI (`gpt-4o-mini`, `whisper-1`, `gpt-4o-mini-tts`)
- Spring Web, Spring Data JPA, MySQL (via Docker Compose)
- Lombok
- JUnit 5 e AssertJ

### Arquitetura (camadas do projeto base, mantidas)

```
domain/          Transaction, Category, TransactionRepository, InvalidTransactionException
application/     Casos de uso (usados pelo REST e como @Tool): Persist, ListByCategory, SummarizeSpending
infrastructure/  http (controller, requests/responses, ApiExceptionHandler) e persistence (JPA)
```

---

## ▶️ Como executar

**Pré-requisitos:** JDK 25, Docker (sobe o MySQL automaticamente pelo `compose.yml`) e uma chave da OpenAI com crédito.

```bash
export OPENAI_API_KEY="sua_chave_aqui"
./gradlew bootRun
```

A aplicação sobe em `http://localhost:8080`. O Spring Boot Docker Compose inicia o MySQL sozinho (porta `3307`).

> No Windows (PowerShell): `$env:OPENAI_API_KEY="sua_chave_aqui"` e `.\gradlew.bat bootRun`.

---

## 🧪 Como testar

### Testes automatizados

Os testes novos **não precisam de OpenAI, banco ou Docker**, porque usam um repositório em memória:

```bash
./gradlew test --tests "dio.budgeting.domain.*" --tests "dio.budgeting.application.*" --tests "dio.budgeting.infrastructure.*"
```

| Teste | O que garante |
|---|---|
| `TransactionTest` | Regras de validação do domínio (valor, descrição, categoria) |
| `PersistTransactionUseCaseTest` | Salva quando válido; **não** salva nada quando inválido |
| `SummarizeSpendingUseCaseTest` | Totais por categoria, total geral e resumo vazio |
| `TransactionOutputTest` | Conversão de centavos para reais |
| `ApiExceptionHandlerTest` | Erro de domínio vira HTTP 400 com a mensagem |

`./gradlew test` (sem filtro) também executa `BudgetingApplicationTests` (precisa de Docker) e os testes `*IT`
de integração com a OpenAI, que só rodam se `OPENAI_API_KEY` estiver definida.

### Fluxo principal pelo REST

```bash
# Criar uma transação (R$ 45,90)
curl -X POST http://localhost:8080/transactions \
  -H "Content-Type: application/json" \
  -d '{"description":"Compra no mercado","category":"GROCERIES","amount":4590}'

# Tentar criar com valor inválido -> deve responder 400
curl -i -X POST http://localhost:8080/transactions \
  -H "Content-Type: application/json" \
  -d '{"description":"Remédio","category":"PHARMA","amount":0}'

# Listar por categoria
curl http://localhost:8080/transactions/GROCERIES

# Resumo dos gastos
curl http://localhost:8080/transactions/summary
```

Formato esperado da resposta do resumo:

```json
{
  "total": 45.9,
  "transactions": 1,
  "byCategory": [
    { "category": "GROCERIES", "total": 45.9, "transactions": 1 },
    { "category": "PHARMA", "total": 0.0, "transactions": 0 },
    { "category": "AUTO", "total": 0.0, "transactions": 0 }
  ]
}
```

### Fluxo principal por voz

Grave um áudio (ou use os exemplos de `src/test/resources/audio/`) e envie:

```bash
curl -X POST http://localhost:8080/transactions/ai \
  -F "file=@src/test/resources/audio/recording-1.m4a" \
  --output resposta.mp3
```

Abra o `resposta.mp3` para ouvir a resposta. Frases que vale a pena testar:

- "Gastei quarenta e cinco reais e noventa centavos no mercado."
- "Quanto eu gastei no total?"
- "Quanto gastei com farmácia?"
- "Gastei zero reais na farmácia." (deve cair na validação)

---

## 📁 O que mudou em relação ao projeto base

| Arquivo | Mudança |
|---|---|
| `domain/Transaction.java` | Validações no construtor de criação |
| `domain/InvalidTransactionException.java` | **Novo**: erro de regra de negócio |
| `domain/TransactionRepository.java` | Novo método `findAll()` |
| `infrastructure/persistence/repository/JpaTransactionRepository.java` | Implementa `findAll()` |
| `application/SummarizeSpendingUseCase.java` | **Novo**: caso de uso + `@Tool summarize-spending` |
| `application/output/*` | `CategoryTotalOutput` e `SpendingSummaryOutput` novos; `TransactionOutput` corrigido (centavos → reais) |
| `infrastructure/http/TransactionController.java` | Registra a nova tool e expõe `GET /transactions/summary` |
| `infrastructure/http/ApiExceptionHandler.java` | **Novo**: erro de domínio vira HTTP 400 |
| `infrastructure/http/response/SpendingSummaryResponse.java` | **Novo** |
| `resources/prompts/system-message.st` | Prompt melhorado |
| `src/test/...` | 5 classes de teste novas + repositório em memória |

---

## 🎓 O que aprendi

> ✏️ **Revise esta seção com suas palavras** antes de entregar. Ela é um rascunho; o ideal é registrar o que
> *você* sentiu, errou e descobriu ao executar o projeto.

- **IA é só mais uma porta de entrada.** A IA chama os mesmos casos de uso que o REST. Por isso as regras de negócio
  (validação, conversão de valores) ficam no domínio e na aplicação, e não no prompt.
- **A descrição do `@Tool` é uma "documentação para o modelo".** É ela que ajuda a IA a decidir qual função chamar.
- **Não deixe o modelo fazer contas.** Os totais são calculados em Java e o modelo apenas os comunica.
- **Prompt para voz é diferente de prompt para texto.** A resposta será falada, então precisa ser curta e sem formatação.
- **Testar sem depender de serviço externo** (repositório em memória) deixa os testes rápidos, baratos e confiáveis.
- **Revisar o projeto base também é aprendizado:** ao ler o código encontrei a inconsistência entre centavos e reais.

---

## 🙌 Créditos

Projeto base e conteúdo das aulas: [DIO Spring Boot Learning Track](https://github.com/digitalinnovationone/dio-spring-boot-learning-track).
Melhorias, testes e documentação desta versão: _seu nome aqui_.
