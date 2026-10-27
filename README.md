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
