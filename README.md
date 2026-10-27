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

**Tipo:** rodada · **Voto:** D

**Justificativa:** Voltar para a versão de terça remove na hora o código em que o `catch` engolia a `SaldoInsuficienteException` e deixava o `debitar(valor + taxa)` rodar sempre, restaurando a regra "saldo ≥ valor + taxa" sem escrever código novo de madrugada. O trade-off é que o rollback desfaz as 3 correções da semana (bugs antigos voltam), não corrige a causa raiz e não cria teste de regressão: a dívida técnica sobe e o conserto de verdade fica para segunda.

**Placar do grupo após esta decisão:** 🔥 6 · 💰 R$ 15 mil · 🧹 3

---

## A busca que não acha ninguém

**Tipo:** relâmpago · **Voto:** B

**Justificativa:** O rollback trouxe de volta o filtro com `==`, que compara referências de `String` e não o conteúdo, por isso a lista vinha vazia. Usar a derived query `findByTitular` do `ContaRepository` (Spring Data JPA) corrige o sintoma e move o filtro para o banco, com parâmetros vinculados, em vez de filtrar conta a conta na aplicação. O trade-off é gastar mais minutos do que o hotfix com `equals` (5 min), em troca de menos dívida técnica e de um endpoint que já nasce protegido contra SQL injection.

**Placar do grupo após esta decisão:** 🔥 6 · 💰 R$ 15 mil · 🧹 2

## A injeção que derreteu o banco

**Tipo:** rodada (desvio por ter escolhido B no ⚡A) · **Voto:** — (sem votação: o desvio se aplicou ao grupo)

**Justificativa:** Como o grupo trocou o filtro pela derived query no ⚡A, o endpoint de busca já usava parâmetros vinculados às 21h02: o valor `' OR '1'='1` é tratado como dado e não como parte do SQL, então a consulta não foi alterada e o ataque morreu na porta (🔥 +1). O trade-off foi ter pago mais tempo no relâmpago em vez do `equals` rápido, o que evitou refazer a busca de madrugada e dispensou soluções frágeis como blocklist.

**Placar do grupo após esta decisão:** 🔥 7 · 💰 R$ 15 mil · 🧹 2
