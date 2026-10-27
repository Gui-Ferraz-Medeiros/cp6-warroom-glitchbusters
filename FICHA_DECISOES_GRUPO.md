# FiapBank – War Room (CP6)

**Grupo:** GlitchBusters
**Integrantes:** Guilherme Ferraz de Medeiros - RM: 564743 | Roberto Marques Moreira - RM: 564935 | Gabriel Jorge Coutinho – RM: 565441 | Felipe Rodrigues Ribeiro dos Santos - RM: 565274 | Manoela Oliveira Bello - RM: 563952

## Dossiê técnico do FiapBank

**Contrato da API**

| Endpoint | Função |
|---|---|
| `POST /api/transferencias` | Realiza uma transferência entre contas (`TransferenciaService.realizar()`), cobrando `valor + taxa` da conta de origem |
| `GET /api/contas/titular/{nome}` | Busca as contas de um titular (`ContaRepository.findByTitular`) |
| `POST /api/contas` | Cria uma conta e devolve o número gerado, ex.: `{"numero": 1001}` |

Representação de uma conta: `{ "titular": "...", "numero": 1001, "saldo": 0.00 }`

**Regras de negócio**

1. Uma transferência só é aceita se `saldo da origem >= valor + taxa`; caso contrário lança `SaldoInsuficienteException` e a transferência é recusada. O saldo nunca pode ficar negativo.
2. O número da conta é um identificador único e sequencial (começa em 1001), gerado pelo `GeradorNumeroConta` (Singleton). Duplicidade é achável em auditoria do Bacen.
3. A busca por titular deve comparar o conteúdo do nome (nunca referências de `String`) e o acesso ao banco deve usar parâmetros vinculados, sem concatenar SQL.
4. Conexões com o banco devem sempre ser fechadas (`try-with-resources`).

**Placar inicial do grupo:** 🔥 7 · 💰 R$ 0 mil · 🧹 2

---

## O saldo que virou negativo

**Tipo:** Rodada 1 · **Voto:** D

**Justificativa:** Voltar para a versão de terça remove na hora o código em que o `catch` engolia a `SaldoInsuficienteException` e deixava o `debitar(valor + taxa)` rodar sempre, restaurando a regra "saldo ≥ valor + taxa" sem escrever código novo de madrugada. O trade-off é que o rollback desfaz as 3 correções da semana (bugs antigos voltam), não corrige a causa raiz e não cria teste de regressão: a dívida técnica sobe e o conserto de verdade fica para segunda.

**Placar do grupo após esta decisão:** 🔥 6 · 💰 R$ 15 mil · 🧹 3

---

## A busca que não acha ninguém

**Tipo:** Evento Relâmpago A · **Voto:** B

**Justificativa:** O rollback trouxe de volta o filtro com `==`, que compara referências de `String` e não o conteúdo, por isso a lista vinha vazia. Usar a derived query `findByTitular` do `ContaRepository` (Spring Data JPA) corrige o sintoma e move o filtro para o banco, com parâmetros vinculados, em vez de filtrar conta a conta na aplicação. O trade-off é gastar mais minutos do que o hotfix com `equals` (5 min), em troca de menos dívida técnica e de um endpoint que já nasce protegido contra SQL injection.

**Placar do grupo após esta decisão:** 🔥 6 · 💰 R$ 15 mil · 🧹 2

## A injeção que derreteu o banco

**Tipo:** Rodada 2 (desvio por ter escolhido B no ⚡A) · **Voto:** — (sem votação: o desvio se aplicou ao grupo)

**Justificativa:** Como o grupo trocou o filtro pela derived query no ⚡A, o endpoint de busca já usava parâmetros vinculados às 21h02: o valor `' OR '1'='1` é tratado como dado e não como parte do SQL, então a consulta não foi alterada e o ataque morreu na porta (🔥 +1). O trade-off foi ter pago mais tempo no relâmpago em vez do `equals` rápido, o que evitou refazer a busca de madrugada e dispensou soluções frágeis como blocklist.

**Placar do grupo após esta decisão:** 🔥 7 · 💰 R$ 15 mil · 🧹 2

---

## A noite das contas gêmeas

**Tipo:** Rodada 3 · **Voto:** A

**Justificativa:** O defeito está no `getInstancia()`, que faz `new GeradorNumeroConta()` a cada chamada e reinicia o contador em 1001, quebrando o Singleton; guardar a instância (`if (instancia == null) instancia = new ...`) restaura a numeração sequencial única em ~30 min e preserva o contrato do número da conta. O trade-off é que não corrige as duplicatas já criadas (exigem auditoria depois), a unicidade continua dependendo da aplicação e não do banco (a sequence seria mais robusta, mas leva ~1 h) e esse `if` simples não é thread-safe sob requisições concorrentes.

**Placar do grupo após esta decisão:** 🔥 8 · 💰 R$ 20 mil · 🧹 2

---

## O comprovante que aponta pra pessoa errada

**Tipo:** Rodada 4 · **Voto:** B

**Justificativa:** Os saldos estavam certos e só o registro do extrato foi gravado com `origem` e `destino` trocados no `new Transferencia(...)`; corrigir a gravação e reprocessar o histórico faz os comprovantes já emitidos voltarem corretos, preservando a integridade do dado que a auditoria vai consultar. O trade-off é ~2h de trabalho de madrugada e mais prejuízo (R$ 25 mil), em troca de não deixar dívida de dados para depois da Black Friday.

**Placar do grupo após esta decisão:** 🔥 9 · 💰 R$ 45 mil · 🧹 1

---

## A fábrica que devolve null

**Tipo:** Rodada 5 · **Voto:** C

**Justificativa:** Trocar o `default -> null` da `ContaFactory` por uma conta CORRENTE elimina o `null` que só explodia lá no `save()` ("Entity must not be null") e faz o cadastro da campanha funcionar na hora. O trade-off é pesado: o tipo inválido deixa de ser detectado e quem pediu UNIVERSITARIA fica com conta corrente (risco de troca de conta, reclamações e cashback perdido); o erro foi escondido em vez de tratado na origem com fail-fast, por isso a dívida técnica sobe.

**Placar do grupo após esta decisão:** 🔥 9 · 💰 R$ 55 mil · 🧹 3

---

## O cadastro com metade dos campos

**Tipo:** Evento Relâmpago C · **Voto:** C

**Justificativa:** Aceitar `null` em CPF e tipo permite que a landing cadastre só com nome e saldo sem mexer na estrutura do construtor, a saída mais rápida para o pedido do marketing. O trade-off é que a invariante da conta (CPF e tipo obrigatórios) deixa de existir: o `null` se espalha e vira `NullPointerException` amanhã, inclusive no `switch (tipo)` da `ContaFactory`, que lança NPE com tipo nulo antes de chegar ao `default`. Um Builder com valores padrão e validação no `build()` teria concentrado isso, mas custava mais tempo na madrugada.

**Placar do grupo após esta decisão:** 🔥 9 · 💰 R$ 55 mil · 🧹 4

---

## O teste vermelho às 03h45

**Tipo:** Rodada 6 · **Voto:** B

**Justificativa:** O teste `tedDeveCobrarTaxaDe5Reais` codifica o contrato com o cliente (TED = R$ 5,00) e o serviço usa o número mágico `50.00`: o vermelho é o sinal correto de que o código está errado. Investigar antes de subir permite corrigir a causa (extrair uma constante com o valor do contrato) em vez de desativar, ignorar ou alterar o teste para confirmar o bug. O trade-off é perder a janela de deploy das 04h, então as correções da noite só entram depois da abertura da campanha (🔥 −1, R$ 30 mil), mas os clientes não são cobrados R$ 50 em vez de R$ 5 na Black Friday.

**Placar do grupo após esta decisão:** 🔥 8 · 💰 R$ 85 mil · 🧹 3

---

## Go/No-Go da Black Friday

**Tipo:** Rodada 7 · **Voto:** B

**Justificativa:** Todo bug da noite passou pelo `TransferenciaService`, uma classe de ~80 linhas que mistura número de conta, taxa, extrato e saldo (responsabilidades demais). Como a suíte está saudável desde a R6, dá para corrigir o que a noite revelou com rede de segurança em vez de abrir com o problema estrutural intacto. O trade-off é abrir 2h atrasado e perder o início do pico (💰 +R$ 100 mil), em troca de reduzir a dívida técnica e evitar que "a correção de segunda nunca chegue".

**Placar do grupo após esta decisão:** 🔥 9 · 💰 R$ 185 mil · 🧹 2

---

## Epílogo: perfil da noite (placar final congelado)

**Placar final:** 🔥 9 · 💰 R$ 185 mil · 🧹 2

**Perfil:** **A Noite dos Heróis** (🔥 7 ou mais e 🧹 até 2): o banco abriu, os clientes confiam e o time dorme (quase) em paz.

---

# Pós-mortem: relatório de incidente da madrugada

**Resumo:** na véspera da Black Friday, o FiapBank teve falhas de regra de negócio (saldo negativo, extrato invertido, taxa errada), de acesso a dados (busca por titular) e de design (numeração de contas, factory e cadastro). O grupo terminou a noite com 🔥 9 · 💰 R$ 185 mil · 🧹 2 e abriu a campanha com a suíte saudável.

## 1. Linha do tempo

| Hora | Incidente | Decisão do grupo | Placar (🔥 · 💰 · 🧹) |
|---|---|---|---|
| 19h12 | R1 · O saldo que virou negativo | D (rollback) | 6 · R$ 15 mil · 3 |
| 20h10 | ⚡A · A busca que não acha ninguém | B (derived query) | 6 · R$ 15 mil · 2 |
| 21h02 | R2 · A injeção que derreteu o banco | desvio (ataque barrado) | 7 · R$ 15 mil · 2 |
| 22h40 | ⚡B · As conexões que não fecham | grupo não afetado | – |
| 22h50 | R3 · A noite das contas gêmeas | A (singleton corrigido) | 8 · R$ 20 mil · 2 |
| 00h47 | R4 · O comprovante que aponta pra pessoa errada | B (corrigir + reprocessar) | 9 · R$ 45 mil · 1 |
| 01h30 | R5 · A fábrica que devolve null | C (conta CORRENTE por padrão) | 9 · R$ 55 mil · 3 |
| 02h20 | ⚡C · O cadastro com metade dos campos | C (aceitar null) | 9 · R$ 55 mil · 4 |
| 03h20 | ⚡D · O teste verde que não testa nada | grupo não afetado | – |
| 03h45 | R6 · O teste vermelho às 03h45 | B (investigar e perder a janela) | 8 · R$ 85 mil · 3 |
| 05h58 | ⚡E · A taxa é 5 ou 50? | grupo não afetado (suíte saudável) | – |
| 06h15 | R7 · Go/No-Go da Black Friday | B (corrigir e abrir 2h atrasado) | 9 · R$ 185 mil · 2 |

## 2. O caminho do grupo

Começamos a noite com um rollback (R1) que apagou o bug do saldo, mas desfez as 3 correções da semana; isso nos levou ao ⚡A, e a escolha pela derived query `findByTitular` acabou nos livrando da injeção de SQL da R2. A R3 e a R4 foram resolvidas atacando a causa (singleton e reprocessamento dos extratos). Na R5 e no ⚡C trocamos o fail-fast por soluções de "fazer funcionar" (tipo padrão e `null` aceito), e a dívida técnica chegou a 4, o pior momento da noite. A R6 virou a chave: investigamos o teste vermelho, descobrimos que o erro estava no código (taxa de R$ 50 contra o contrato de R$ 5) e isso deixou a suíte saudável para o Go/No-Go, onde optamos por corrigir antes de abrir.

## 3. Causa raiz de 2 incidentes

**R1 · O saldo que virou negativo (Aula 11, exceções).** A `SaldoInsuficienteException` era lançada dentro de um `try` e capturada pelo próprio método por um `catch` vazio ("cliente especial"), então a exceção nunca chegava ao chamador. Como o `origem.debitar(valor + taxa)` ficava depois do bloco, ele rodava sempre e o saldo ficava negativo. O mecanismo é a **exceção engolida no mesmo método em que foi lançada**, com uma regra de negócio ("cliente especial") escondida num `catch` vazio. A correção certa é deixar a exceção propagar e registrar um teste com `assertThrows`; a exceção para clientes especiais deveria ser uma regra explícita (limite especial), não um `catch` vazio.

**R3 · A noite das contas gêmeas (Aula 14, Singleton).** O `getInstancia()` fazia `return new GeradorNumeroConta()`, sem campo estático guardando a instância; cada chamada criava um objeto novo cujo contador recomeçava em 1001. O mecanismo é um **Singleton sem instância guardada**. Um Singleton exige construtor privado, campo estático e um `getInstancia()` que só cria a instância quando ela é `null` (de forma segura para concorrência). A solução definitiva seria delegar a numeração ao banco (sequence), que garante a unicidade independentemente da aplicação.

## 4. O que faríamos diferente

**R5 · A fábrica que devolve null (escolhemos C → faríamos A).** Devolver uma conta CORRENTE por padrão escondeu o erro e criou contas de tipo errado. Com fail-fast, a `ContaFactory` lançaria `IllegalArgumentException("Tipo de conta inválido: UNIVERSITARIA")` na origem do problema, e o controller traduziria isso em uma resposta HTTP 400 clara para o cliente. O custo seria pequeno (~30 min) perto de contas trocadas, reclamações e cashback perdido.

**⚡C · O cadastro com metade dos campos (escolhemos C → faríamos B).** Aceitar `null` nos campos espalhou o risco de `NullPointerException` (inclusive no `switch (tipo)` da factory). Um Builder com valores padrão e validação no `build()` concentraria as regras de cadastro em um lugar só e manteria a invariante da conta (CPF e tipo obrigatórios).

## 5. Uma palavra para a madrugada

**Contrato.**
