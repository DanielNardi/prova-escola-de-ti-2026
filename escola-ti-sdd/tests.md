# Casos de Teste — Zona Azul Digital API

Variante: `TARIFA=400¢/h`, `FRAÇÃO=30min`, `TETO=5000¢`, `TOLERÂNCIA=15min`, `PORTA=8001`

> [!WARNING]
> **Regra do teto:** `valor_centavos` nunca supera `5000`. Qualquer cálculo que resulte em valor maior **deve ser truncado para 5000**, não retornado como calculado. O agente gerador deve verificar esta regra antes de retornar a resposta do UC2.

---

## Regra 1 — Fração de cobrança (`FRACAO_MINUTOS = 30`)

Fórmula: `fracoes = ceil(minutos / 30)`, `valor = fracoes × 200¢`

Estes casos assumem duração **acima** de `TOLERANCIA_MINUTOS = 15` (caso contrário, valor é 0 — ver Regra 2).

| Cenário | `minutos` | `fracoes` | `valor_centavos` esperado |
|---|---|---|---|
| Exatamente 1 fração | 30 | 1 | 200 |
| Adjacência: 1 minuto além da fração | 31 | 2 | 400 |
| Exatamente 2 frações | 60 | 2 | 400 |
| Adjacência: 1 minuto além de 2 frações | 61 | 3 | 600 |
| Primeiro minuto faturável (logo acima da tolerância) | 16 | 1 | 200 |
| 29 minutos (quase 1 fração, acima da tolerância) | 29 | 1 | 200 |

> [!WARNING]
> Fração **exata** (30 min) conta como 1 fração, não como 0. Implementações que usam `//` em vez de `ceil` erram neste caso.

> [!WARNING]
> Os casos `minutos = 1` e `minutos = 14` (abaixo da tolerância de 15min) **não devem chegar ao cálculo de frações** — retornam `valor_centavos: 0` diretamente pela regra da tolerância. Não misture as duas regras.

---

## Regra 2 — Tolerância gratuita (`TOLERANCIA_MINUTOS = 15`)

A tolerância **não** é descontada — ultrapassar por 1 minuto cobra integral desde o minuto 1.

| Cenário | `minutos` | `valor_centavos` esperado | Observação |
|---|---|---|---|
| Dentro da tolerância (exato) | 15 | 0 | Limite exato = grátis |
| 1 minuto além da tolerância | 16 | 200 | Cobra `ceil(16/30)=1` fração |
| Tolerância zero (0 min) | 0 | 0 | Entrada e saída simultâneas |
| 14 minutos | 14 | 0 | Abaixo do limite |

> [!WARNING]
> Ultrapassar a tolerância por 1 minuto não cobra apenas 1 minuto — cobra a **fração inteira** desde o início. `16 min → ceil(16/30) = 1 fração → 200¢`, não `ceil(1/30) × 200¢`.

---

## Regra 3 — Teto diário (`TETO_DIARIO_CENTAVOS = 5000`)

O teto é aplicado **após** o cálculo de frações, como último passo antes de retornar.

| Cenário | `minutos` | Valor bruto calculado | `valor_centavos` esperado |
|---|---|---|---|
| Exatamente no teto | 750 | `ceil(750/30) × 200 = 5000` | 5000 |
| 1 fração além do teto | 780 | `ceil(780/30) × 200 = 5200` | 5000 |
| Muito além do teto | 1440 (24h) | `ceil(1440/30) × 200 = 9600` | 5000 |
| Abaixo do teto | 60 | `ceil(60/30) × 200 = 400` | 400 |
| Dentro da tolerância (teto irrelevante) | 15 | — | 0 |

> [!WARNING]
> O teto se aplica ao resultado **bruto de frações**, não à tarifa horária. Um bilhete de 750 minutos gera exatamente 5000¢ (no teto), e 751 minutos gera `ceil(751/30) × 200 = 26 × 200 = 5200¢` → truncado para **5000¢**.

---

## Regra 4 — Arredondamento do tempo médio no relatório (half-up)

| Cenário | Durações dos bilhetes encerrados | Média exata | `tempo_medio_minutos` esperado |
|---|---|---|---|
| Média exata inteira | [40, 60] | 50,0 | 50 |
| Média com 0,5 (half-up → cima) | [46, 47] | 46,5 | 47 |
| Média com 0,4 (trunca) | [46, 46] | 46,0 | 46 |
| Média com 0,6 (arredonda) | [46, 47, 48] | 47,0 | 47 |
| 1 bilhete apenas | [33] | 33,0 | 33 |
| Sem bilhetes encerrados | [] | — | 0 |

> [!WARNING]
> Python `round()` usa **half-even** (bancário): `round(46.5) = 46`, não 47. Use `math.floor(media + 0.5)` para obter half-up correto.

---

## Regra 5 — Uma vaga por placa (UC8)

| Cenário | Ação | Resposta esperada |
|---|---|---|
| Placa sem histórico | `POST /bilhetes {"placa": "AAA0000"}` | 201 |
| Placa com bilhete aberto | `POST /bilhetes {"placa": "AAA0000"}` (2ª vez) | 409 `bilhete_em_aberto` |
| Placa após encerramento | `POST /bilhetes/1/encerramento` + `POST /bilhetes {"placa": "AAA0000"}` | 201 |
| Placa após cancelamento | `POST /bilhetes/1/cancelamento` + `POST /bilhetes {"placa": "AAA0000"}` | 201 |

---

## Regra 6 — Validação de placa

| Entrada | Status esperado | Código de erro |
|---|---|---|
| `"ABC1D23"` (7 alnum maiúsc.) | 201 | — |
| `"abc1d23"` (minúsculas) | 422 | `placa_invalida` |
| `"ABC-1D23"` (hífen) | 422 | `placa_invalida` |
| `"ABC1D2"` (6 chars) | 422 | `placa_invalida` |
| `"ABC1D234"` (8 chars) | 422 | `placa_invalida` |
| `""` (vazio) | 422 | `placa_invalida` |
| ausente no body | 422 | `placa_invalida` |

---

## Regra 7 — Transições de estado

| Estado atual | Ação | Resposta esperada |
|---|---|---|
| `aberto` | encerrar | 200, `status: "encerrado"` |
| `aberto` | cancelar | 200, `status: "cancelado"` |
| `encerrado` | encerrar novamente | 409 `bilhete_ja_encerrado` |
| `cancelado` | encerrar | 409 `bilhete_ja_encerrado` |
| `encerrado` | cancelar | 409 `bilhete_nao_aberto` |
| `cancelado` | cancelar novamente | 409 `bilhete_nao_aberto` |
| qualquer | encerrar/cancelar com id inválido | 404 `bilhete_nao_encontrado` |

---

## Regra 8 — Relatório diário: filtro por data

| Cenário | Configuração | `total_bilhetes` esperado |
|---|---|---|
| Data sem bilhetes | Nenhum encerramento na data | 0 |
| `saida` no dia consultado | 3 encerramentos em `2026-10-05` | 3 |
| `saida` em outro dia | Encerramentos em `2026-10-04` | 0 para `?data=2026-10-05` |
| Bilhetes cancelados no dia | Cancelamentos em `2026-10-05` | 0 (cancelados excluídos) |
| Bilhetes ainda abertos | Abertura em `2026-10-05`, sem encerramento | 0 |
| Data inválida | `?data=05-10-2026` | 422 `data_invalida` |

---

## Regra 9 — Campo `entrada` opcional no UC1

| Cenário | `entrada` enviado | Comportamento esperado |
|---|---|---|
| Ausente | — | `entrada` = instante atual do servidor (offset `-03:00`) |
| Válido ISO-8601 com offset | `"2026-10-01T08:00:00-03:00"` | `entrada` = valor enviado, preservado exatamente |
| Inválido (texto livre) | `"ontem"` | 422 `entrada_invalida` |
| Sem offset (naive datetime) | `"2026-10-01T08:00:00"` | 422 `entrada_invalida` (offset obrigatório) |
| Formato de data sem hora | `"2026-10-01"` | 422 `entrada_invalida` |
| Offset UTC (`Z`) | `"2026-10-01T11:00:00Z"` | 422 `entrada_invalida` — apenas `-03:00` é aceito |

---

## Regra 10 — Integridade de tipos nos campos numéricos

Todos os campos numéricos da API devem ser `int`, nunca `float`. Um agente que use `/` em Python pode produzir `float` inadvertidamente.

| Campo | Endpoint | Tipo esperado | Armadilha comum |
|---|---|---|---|
| `valor_centavos` | UC2 | `int` | divisão `/` em vez de `//` |
| `minutos` | UC2 | `int` | `timedelta.seconds / 60` retorna `float` |
| `total_bilhetes` | UC4 | `int` | `len()` já retorna `int` — sem risco |
| `faturamento_centavos` | UC4 | `int` | `sum()` de `int` já é `int` — sem risco |
| `tempo_medio_minutos` | UC4 | `int` | `statistics.mean()` retorna `float` — deve converter |
