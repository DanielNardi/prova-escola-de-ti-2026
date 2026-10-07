# Tarefas de Implementação — Zona Azul Digital API

Stack: Python 3.12 + Flask + Gunicorn. Decomposição em tarefas ordenadas por dependência.

---

## Tarefa 1 — Estrutura do projeto e configuração

**Objetivo:** criar o esqueleto da aplicação com dependências, constantes da variante e modelos de dados.

**Entregáveis:**
- `requirements.txt` com `flask` e `gunicorn` em versões fixas (pinadas).
- `app/config.py` com as constantes: `TARIFA_HORA_CENTAVOS=400`, `FRACAO_MINUTOS=30`, `TETO_DIARIO_CENTAVOS=5000`, `TOLERANCIA_MINUTOS=15`, `PORTA_SERVICO=8001`.
- `app/models.py` com dataclasses ou dicts tipados: `Bilhete` (campos: `id`, `placa`, `entrada`, `status`, `saida?`, `minutos?`, `valor_centavos?`), `RelatorioDiario`.
- `app/storage.py` com repositório em memória: `dict[int, dict]` e contador `proximo_id`, iniciando em 1.
- `app/__init__.py` com factory `criar_app()` — instância Flask, registro de blueprints, handlers de erro globais.
- `wsgi.py` ou `run.py` que chama `criar_app()` e sobe Gunicorn em `0.0.0.0:8001`.

**Critério de conclusão:** `flask run --port 8001` (ou `gunicorn`) sobe sem erro; `GET /bilhetes/ativos` retorna 200 com `[]`.

---

## Tarefa 2 — Lógica de negócio (cálculos e validações)

**Objetivo:** implementar as funções puras de cálculo e validação, testáveis sem HTTP.

**Entregáveis:**
- `app/calculos.py` com:
  - `calcular_minutos(entrada: datetime, saida: datetime) -> int` — duração em minutos inteiros (`ceil(segundos / 60)`).
  - `calcular_valor(minutos: int) -> int` — aplica tolerância, frações e teto; retorna centavos inteiros.
  - `calcular_tempo_medio(duracoes: list[int]) -> int` — média com arredondamento half-up (`floor(media + 0.5)`).
- `app/validacoes.py` com:
  - `validar_placa(placa: str) -> bool` — regex `^[A-Z0-9]{7}$`.
  - `validar_data(data: str) -> bool` — formato `AAAA-MM-DD`.
  - `validar_entrada_datetime(valor: str) -> datetime | None` — parse ISO-8601 com offset obrigatório; retorna `None` se inválido.

**Verificação obrigatória antes de avançar:**

| Chamada | Resultado esperado |
|---|---|
| `calcular_valor(15)` | `0` |
| `calcular_valor(16)` | `200` |
| `calcular_valor(30)` | `200` |
| `calcular_valor(31)` | `400` |
| `calcular_valor(750)` | `5000` |
| `calcular_valor(780)` | `5000` |
| `calcular_tempo_medio([46, 47])` | `47` |
| `calcular_tempo_medio([])` | `0` |

**Critério de conclusão:** todos os casos acima retornam os valores esperados.

---

## Tarefa 3 — Endpoints de bilhetes (UC1, UC2, UC3, UC5, UC6, UC8)

**Objetivo:** implementar o blueprint `/bilhetes` com todos os casos de uso de bilhete.

**Entregáveis:**
- `app/routes/bilhetes.py` — Blueprint com prefixo `/bilhetes`:
  - `POST /bilhetes` — UC1 (abrir bilhete) + UC8 (rejeitar placa duplicada aberta).
  - `POST /bilhetes/<int:id>/encerramento` — UC2 (encerrar e calcular valor).
  - `GET /bilhetes/ativos` — UC3 (listar abertos, ordenado por `entrada` desc).
  - `POST /bilhetes/<int:id>/cancelamento` — UC5 (cancelar bilhete aberto).
  - `GET /bilhetes` — UC6 (histórico por placa via query param `?placa=`).
- Registro do blueprint em `app/__init__.py`.
- Atenção ao roteamento: `GET /bilhetes/ativos` deve ser registrado **antes** de `GET /bilhetes/<int:id>` para evitar conflito de rota no Flask.

**Critério de conclusão:** todos os critérios de aceite das UCs 1, 2, 3, 5, 6 e 8 em `specs.md` passam quando testados com `curl` ou cliente HTTP.

---

## Tarefa 4 — Endpoint de relatório (UC4)

**Objetivo:** implementar o blueprint `/relatorios` com o relatório diário.

**Entregáveis:**
- `app/routes/relatorios.py` — Blueprint com:
  - `GET /relatorios/diario` — UC4 (aceita `?data=AAAA-MM-DD`, filtra bilhetes encerrados no dia, agrega totais).
- Registro do blueprint em `app/__init__.py`.
- `tempo_medio_minutos` usa `math.floor(media + 0.5)` — **não** `round()` nativo do Python (half-even).

**Critério de conclusão:** todos os critérios de aceite da UC4 em `specs.md` passam, incluindo `tempo_medio_minutos` com half-up e exclusão de bilhetes cancelados.

---

## Tarefa 5 — Docker e validação de contrato

**Objetivo:** containerizar a aplicação e garantir conformidade total com o contrato.

**Entregáveis:**
- `Dockerfile` (ou `Containerfile`) com imagem Python, instalação de dependências e `CMD gunicorn -w 1 -b 0.0.0.0:8001 "app:criar_app()"`.
- `compose.yaml` (ou `docker-compose.yml`) mapeando porta `8001:8001`.
- Checklist de conformidade antes de declarar conclusão:
  - [ ] Nenhum campo retorna `float` onde `int` é exigido (`valor_centavos`, `minutos`, `tempo_medio_minutos`, `faturamento_centavos`, `total_bilhetes`).
  - [ ] Todos os timestamps têm offset `-03:00`.
  - [ ] Envelope de erro é sempre `{"erro": "<codigo>"}`, sem chaves extras.
  - [ ] API escuta em `http://localhost:8001`.
  - [ ] `GET /bilhetes/ativos` e `GET /bilhetes?placa=` retornam `[]` (não `null`) quando sem resultados.
  - [ ] Bilhetes `"aberto"` e `"cancelado"` no UC6 não retornam `saida`, `minutos` nem `valor_centavos`.

**Critério de conclusão:** `docker compose up -d --build` sobe sem erro; `GET http://localhost:8001/bilhetes/ativos` retorna HTTP 200 com `[]`.
