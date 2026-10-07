# Plano Técnico — Zona Azul Digital API

## 1. Stack

**Decisão:** Python 3.12 + Flask + Gunicorn

**Justificativa:**
- `constitution.md` determina Flask como framework. Flask é leve, sem magia de validação automática, o que torna o comportamento de erros 100% controlável — importante para garantir o envelope `{"erro": "..."}` em todos os cenários.
- Gunicorn serve a aplicação em produção; Flask built-in server é usado apenas em desenvolvimento/testes.
- Python tem biblioteca padrão `math.ceil` e `datetime` com suporte completo a timezones — ambos necessários para a lógica de fração e fuso `-03:00`.
- Dependências mínimas: `flask`, `gunicorn` — sem banco de dados externo.

## 2. Persistência

**Decisão:** 
armazenamento em memória (dicionário Python em escopo de módulo)

**Justificativa:**
- O enunciado exige apenas uma API, sem requisito de durabilidade entre reinicializações.
- Memória elimina dependência de banco de dados, simplifica deploy e garante estado limpo entre execuções de teste.
- Estrutura: `dict[int, Bilhete]` onde a chave é o `id` sequencial. Acesso O(1) por `id`; varredura para filtros por placa é O(n) aceitável para o volume esperado.

## 3. Relógio e fuso horário

**Decisão:** `datetime.now(tz=datetime.timezone(timedelta(hours=-3)))` para capturar o instante atual com offset fixo `-03:00`.

**Justificativa:**
- O contrato fixa o offset `-03:00` de forma absoluta. Usar `ZoneInfo("America/Sao_Paulo")` introduziria variação durante horário de verão (quando o offset vira `-02:00`), quebrando a conformidade. Por isso usa-se `datetime.timezone(timedelta(hours=-3))` — offset fixo, sem ambiguidade de DST.
- O campo `entrada` opcional (UC1) permite injetar timestamps arbitrários nos testes sem depender de tempo real — o serviço aceita e armazena o valor enviado sem validação de "não pode ser futuro".
- O parse de `entrada` recebida pelo cliente deve exigir que o valor já contenha offset (ISO-8601 com timezone info). Strings sem offset são rejeitadas com 422.

## 4. Lógica de cálculo de valor

**Decisão:** implementar em função pura isolada, com as constantes da variante como parâmetros

```python
import math

TARIFA_HORA_CENTAVOS = 400
FRACAO_MINUTOS = 30
TETO_DIARIO_CENTAVOS = 5000
TOLERANCIA_MINUTOS = 15

def calcular_valor(minutos: int) -> int:
    if minutos <= TOLERANCIA_MINUTOS:
        return 0
    fracoes = math.ceil(minutos / FRACAO_MINUTOS)
    valor_por_fracao = TARIFA_HORA_CENTAVOS // (60 // FRACAO_MINUTOS)
    valor_bruto = fracoes * valor_por_fracao
    return min(valor_bruto, TETO_DIARIO_CENTAVOS)
```

**Justificativa:**
- Divisão inteira `//` para `valor_por_fracao` garante que o resultado seja inteiro sem ponto flutuante intermediário.
- `math.ceil` implementa "arredondamento para cima" exigido pelo contrato.
- Função pura facilita teste unitário isolado.

## 5. Validação de placa

**Decisão:** regex `^[A-Z0-9]{7}$` aplicada manualmente na função `validar_placa` antes de qualquer operação que receba placa como parâmetro.

**Justificativa:** Flask não faz validação automática de body — toda validação é explícita no código. Regex é a forma mais direta e auditável de expressar a regra do contrato (7 alfanuméricos maiúsculos).

## 6. Serialização de datas

**Decisão:** serializar todos os `datetime` com `isoformat()` mantendo o offset `-03:00`

**Justificativa:** ISO-8601 com offset é o formato exigido pelo contrato. `datetime` com `tzinfo` configurado para `-03:00` serializa corretamente via `isoformat()`.

## 7. Porta do serviço

**Decisão:** `PORTA_SERVICO = 8001` lida de variável de ambiente `PORT` com fallback para `8001`

**Justificativa:** permite sobrescrever sem alterar código, mas garante o valor correto da variante por padrão.

## 8. Estrutura de arquivos sugerida

```
app/
  __init__.py      # factory Flask (create_app), registro de blueprints, handlers de erro
  config.py        # constantes da variante (sem imports internos)
  models.py        # dataclasses ou dicts tipados para Bilhete e Relatorio
  validacoes.py    # funções puras de validação — importa config.py
  calculos.py      # lógica de valor e tempo (funções puras) — importa config.py
  storage.py       # repositório em memória — importa models.py
  routes/
    bilhetes.py    # Blueprint /bilhetes — UC1, UC2, UC3, UC5, UC6, UC8
    relatorios.py  # Blueprint /relatorios — UC4
```

Ordem de dependência (sem ciclos):

```
config.py  ←  calculos.py
config.py  ←  validacoes.py
config.py  ←  models.py
models.py  ←  storage.py
storage.py + calculos.py + validacoes.py  ←  routes/bilhetes.py
storage.py + calculos.py                 ←  routes/relatorios.py
routes/*  ←  app/__init__.py
```

## 9. Tratamento de erros

**Decisão:** usar `flask.abort()` com código HTTP e registrar handlers com `@app.errorhandler` para garantir que **toda** resposta de erro siga o envelope `{"erro": "<codigo>"}`.

**Justificativa:** Flask não tem validação automática — cada endpoint valida explicitamente e chama `abort()` ou retorna `jsonify({"erro": "..."})` diretamente. Um handler para código 404 e 409 garante formato consistente mesmo para erros não antecipados.

## 10. `tempo_medio_minutos` no relatório

**Decisão:** usar `round(media, 0)` com lógica half-up explícita via `math.floor(media + 0.5)`

**Justificativa:** o `round()` nativo do Python 3 usa half-even (bancário), que não é o exigido. `math.floor(media + 0.5)` implementa half-up fielmente.
