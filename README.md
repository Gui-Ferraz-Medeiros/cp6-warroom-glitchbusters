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
