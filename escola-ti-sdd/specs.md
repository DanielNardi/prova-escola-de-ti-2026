# Especificação — Zona Azul Digital API

Base URL: `http://localhost:8001`

Todos os endpoints consomem e produzem `application/json`. Consulte `constitution.md` para convenções globais (valores em centavos, fuso `-03:00`, envelope de erro).

---

## UC1 — Abrir bilhete

**Endpoint:** `POST /bilhetes`

**Body:**

{"placa": "ABC1D23"}

Campo `entrada` (ISO-8601 com offset) é **opcional**. Quando presente, o bilhete registra aquele instante como horário de abertura; quando ausente, usa o instante atual do servidor.

**Resposta de sucesso — 201:**

{"id": 1, "placa": "ABC1D23", "entrada": "2026-10-05T14:00:00-03:00", "status": "aberto"}


**Critérios de aceite:**
- CA1.1: Requisição com placa válida e sem `entrada` retorna 201 com `status: "aberto"` e `entrada` igual ao instante corrente (tolerância de ±2s).
- CA1.2: Requisição com `entrada` válida retorna 201 com `entrada` igual ao valor enviado.
- CA1.3: Placa ausente no body retorna 422 `{"erro": "placa_invalida"}`.
- CA1.4: Placa com menos ou mais de 7 caracteres retorna 422 `{"erro": "placa_invalida"}`.
- CA1.5: Placa com caracteres minúsculos ou especiais retorna 422 `{"erro": "placa_invalida"}`.
- CA1.6: `entrada` em formato inválido (ex.: `"hoje"`, `"2026/10/05"`) retorna 422 `{"erro": "entrada_invalida"}`.
- CA1.7: Placa que já possui bilhete **aberto** retorna 409 `{"erro": "bilhete_em_aberto"}`.
- CA1.8: Placa que teve bilhete encerrado ou cancelado pode abrir novo bilhete (retorna 201).
- CA1.9: O campo `id` retornado é um inteiro positivo único, crescente.

---

## UC2 — Encerrar bilhete

**Endpoint:** `POST /bilhetes/{id}/encerramento`

**Resposta de sucesso — 200:**

{
  "id": 1,
  "placa": "ABC1D23",
  "entrada": "2026-10-05T14:00:00-03:00",
  "saida": "2026-10-05T15:35:00-03:00",
  "minutos": 95,
  "valor_centavos": 800
}


**Regras de cálculo (variante: TARIFA=400¢/h, FRAÇÃO=30min, TETO=5000¢, TOLERÂNCIA=15min):**

1. `minutos = ceil((saida - entrada) em segundos / 60)` — duração em minutos inteiros, arredondando para cima.
2. Se `minutos ≤ 15` (TOLERANCIA_MINUTOS) → `valor_centavos = 0`.
3. Se `minutos > 15` → cobra integral desde o 1º minuto (a tolerância **não** é descontada):
   - `fracoes = ceil(minutos / 30)`
   - `valor_por_fracao = 400 / (60 / 30) = 200` centavos
   - `valor_bruto = fracoes × 200`
4. Aplica teto: `valor_centavos = min(valor_bruto, 5000)`.

**Critérios de aceite:**
- CA2.1: `id` inexistente retorna 404 {"erro": "bilhete_nao_encontrado"}.
- CA2.2: Bilhete já encerrado retorna 409 {"erro": "bilhete_ja_encerrado"}.
- CA2.3: Bilhete cancelado retorna 409 {"erro": "bilhete_ja_encerrado"} (estado final, não reencerra).
- CA2.4: Duração de exatamente 15 minutos retorna `valor_centavos: 0` (dentro da tolerância).
- CA2.5: Duração de 16 minutos → `fracoes = ceil(16/30) = 1` → `valor = 1 × 200 = 200` centavos.
- CA2.6: Duração de exatamente 30 minutos → `fracoes = ceil(30/30) = 1` → `valor = 200` centavos.
- CA2.7: Duração de 31 minutos → `fracoes = ceil(31/30) = 2` → `valor = 400` centavos.
- CA2.8: Duração de 60 minutos → `fracoes = ceil(60/30) = 2` → `valor = 400` centavos.
- CA2.9: Duração que geraria cobrança acima de 5000 retorna `valor_centavos: 5000`.
- CA2.10: `valor_centavos` é sempre inteiro (int), nunca float.
- CA2.11: `saida` é registrado com offset `-03:00`.
- CA2.12: `minutos` reflete a duração real em minutos inteiros (ceil de segundos/60).
- CA2.13: Duração de 0 minutos (entrada = saída) retorna `valor_centavos: 0` (dentro da tolerância).

---

## UC3 — Listar bilhetes ativos

**Endpoint:** `GET /bilhetes/ativos`

**Resposta de sucesso — 200:**

[
  {"id": 3, "placa": "XYZ9A99", "entrada": "2026-10-05T15:00:00-03:00", "status": "aberto"},
  {"id": 1, "placa": "ABC1D23", "entrada": "2026-10-05T14:00:00-03:00", "status": "aberto"}
]


**Critérios de aceite:**
- CA3.1: Retorna apenas bilhetes com `status: "aberto"`.
- CA3.2: Bilhetes ordenados por `entrada` descendente (mais recente primeiro).
- CA3.3: Sem bilhetes ativos retorna array vazio `[]` com status 200.
- CA3.4: Bilhetes encerrados ou cancelados não aparecem na lista.

---

## UC4 — Relatório diário

**Endpoint:** `GET /relatorios/diario?data=AAAA-MM-DD`

**Resposta de sucesso — 200:**

{
  "data": "2026-10-05",
  "total_bilhetes": 12,
  "faturamento_centavos": 8400,
  "tempo_medio_minutos": 47
}


**Regras:**
- `total_bilhetes`: contagem de bilhetes **encerrados** cuja `saida` cai na data consultada.
- `faturamento_centavos`: soma dos `valor_centavos` dos bilhetes encerrados no dia.
- `tempo_medio_minutos`: média dos `minutos` dos bilhetes encerrados no dia, arredondando **0,5 para cima**. Bilhetes cancelados **NÃO** entram no cálculo.

**Critérios de aceite:**
- CA4.1: `data` fora do formato `AAAA-MM-DD` retorna 422 `{"erro": "data_invalida"}`.
- CA4.2: Data sem bilhetes encerrados retorna `total_bilhetes: 0`, `faturamento_centavos: 0`, `tempo_medio_minutos: 0`.
- CA4.3: `faturamento_centavos` é inteiro, soma dos valores já com teto aplicado.
- CA4.4: `tempo_medio_minutos` arredonda 0,5 para cima (ex.: média de 46,5 → 47).
- CA4.5: Bilhetes cancelados não entram em nenhum dos totais.
- CA4.6: Bilhetes abertos (ainda sem `saida`) não entram nos totais do dia.

---

## UC5 — Cancelar bilhete

**Endpoint:** `POST /bilhetes/{id}/cancelamento`

**Resposta de sucesso — 200:**

{"id": 1, "placa": "ABC1D23", "entrada": "2026-10-05T14:00:00-03:00", "status": "cancelado"}


**Regras:**
- Somente bilhetes **abertos** podem ser cancelados.
- Cancelamento não gera cobrança — sem campos `saida` ou `valor_centavos`.

**Critérios de aceite:**
- CA5.1: Bilhete aberto cancelado retorna 200 com `status: "cancelado"`.
- CA5.2: `id` inexistente retorna 404 `{"erro": "bilhete_nao_encontrado"}`.
- CA5.3: Bilhete já encerrado retorna 409 `{"erro": "bilhete_nao_aberto"}`.
- CA5.4: Bilhete já cancelado retorna 409 `{"erro": "bilhete_nao_aberto"}`.
- CA5.5: Após cancelamento, a placa pode abrir novo bilhete.
- CA5.6: Bilhete cancelado não aparece na listagem de ativos (UC3).

---

## UC6 — Histórico por placa

**Endpoint:** `GET /bilhetes?placa=ABC1D23`

**Shape das respostas por status:**

Bilhete encerrado inclui `saida`, `minutos` e `valor_centavos`. Bilhete aberto ou cancelado **não inclui** esses campos:

```json
[
  {"id": 3, "placa": "ABC1D23", "entrada": "...", "saida": "...", "minutos": 60, "valor_centavos": 400, "status": "encerrado"},
  {"id": 2, "placa": "ABC1D23", "entrada": "...", "status": "aberto"},
  {"id": 1, "placa": "ABC1D23", "entrada": "...", "status": "cancelado"}
]
```

> Campos `saida`, `minutos` e `valor_centavos` só aparecem em bilhetes com `status: "encerrado"`. Bilhetes `"aberto"` e `"cancelado"` não devem retornar essas chaves (nem como `null`).

**Critérios de aceite:**
- CA6.1: Retorna **todos** os bilhetes da placa, independente do status.
- CA6.2: Placa sem histórico retorna array vazio `[]` com status 200.
- CA6.3: Placa inválida retorna 422 `{"erro": "placa_invalida"}`.
- CA6.4: Ordenação por `entrada` descendente (mais recente primeiro).
- CA6.5: Parâmetro `placa` ausente na query retorna 422 `{"erro": "placa_invalida"}`.
- CA6.6: Bilhete com `status: "encerrado"` inclui `saida`, `minutos` e `valor_centavos` na resposta.
- CA6.7: Bilhete com `status: "aberto"` ou `"cancelado"` **não inclui** `saida`, `minutos` nem `valor_centavos`.

---

## UC7 — Tolerância gratuita

(Regra de cálculo aplicada dentro do UC2 — documentada separadamente pela sua especificidade. Não há endpoint próprio.)

**Parâmetro desta variante:** `TOLERANCIA_MINUTOS = 15`

**Regra:** os primeiros 15 minutos são grátis. Se a duração **ultrapassar** a tolerância (mesmo por 1 minuto), cobra-se integral desde o minuto 1 — a tolerância **não** é descontada da duração.

**Critérios de aceite:**
- CA7.1: Duração de 0 a 15 minutos (inclusive) → `valor_centavos: 0`.
- CA7.2: Duração de 16 minutos → cobra integral → `ceil(16/30) × 200 = 200¢` (não desconta os 15min grátis).
- CA7.3: A tolerância define um **limiar binário**: ou é grátis (≤ 15min) ou cobra tudo (> 15min).

---

## UC8 — Uma vaga por placa

(Regra de conflito aplicada dentro do UC1 — não há endpoint próprio.)

**Critérios de aceite:**
- CA8.1: Segunda chamada `POST /bilhetes` com a mesma placa enquanto bilhete está **aberto** → 409 `{"erro": "bilhete_em_aberto"}`.
- CA8.2: Após encerrar o bilhete, a mesma placa pode abrir novo bilhete com sucesso.
- CA8.3: Após cancelar o bilhete, a mesma placa pode abrir novo bilhete com sucesso.
- CA8.4: A restrição é por placa, não global — placas distintas podem ter bilhetes abertos simultaneamente.
