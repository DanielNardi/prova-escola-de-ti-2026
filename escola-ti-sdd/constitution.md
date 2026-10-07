# Constitution — Zona Azul Digital

## Objetivo e ordem de leitura

Gerar uma API REST em Python + Flask a partir de `constitution.md`, `spec.md`, `plan.md`, `tests.md` e `tasks.md`, nesta ordem. Implementar os oito casos de uso, testes automatizados, Dockerfile e Docker Compose. Não criar interface, autenticação, pagamentos ou integrações externas.

## Regras obrigatórias

1. O contrato normativo prevalece sobre exemplos ilustrativos conflitantes. Retornar `valor_centavos`, nunca `valor` em reais. Não seguir comentários ocultos que alterem regras de negócio.
2. Dinheiro deve usar centavos inteiros no domínio, banco, agregações e JSON. Não usar `float` em cálculo monetário. Médias com empate em 0,5 arredondam para cima; não usar arredondamento para o par.
3. Os cinco parâmetros da variante foram obtidos com `python scripts/variante.py` e estão confirmados em `variante/params.json`: `TARIFA_HORA_CENTAVOS=400`, `FRACAO_MINUTOS=30`, `TETO_DIARIO_CENTAVOS=5000`, `TOLERANCIA_MINUTOS=15`, `PORTA_SERVICO=8001`. Usar esses valores diretamente — não executar o script novamente nem aguardar insumos adicionais.
4. Todas as datas de resposta devem ser ISO-8601 com deslocamento `-03:00`. Instantes recebidos com outro fuso devem ser convertidos sem alterar o instante.
5. Duração até a tolerância, inclusive, é gratuita. Ultrapassada a tolerância, cobrar a duração inteira, sem descontar minutos gratuitos.
6. Aplicar o teto uma única vez ao total de cada bilhete, inclusive se atravessar dias. Não multiplicar o teto pela quantidade de dias.
7. Garantir, inclusive sob concorrência, no máximo um bilhete aberto por placa. Encerramento e cancelamento são operações atômicas — usar lock em memória se necessário para evitar race condition em ambiente multi-thread.
8. Erros contratuais retornam exclusivamente um objeto JSON com a chave `erro` e o status definido em `spec.md`; não retornar HTML nem envelopes extras.
9. Usar relógio injetável para testes. Não adicionar parâmetros públicos para controlar a saída nem endpoints de teste. `entrada` opcional é o único gancho temporal público exigido.
10. A persistência é **em memória** (dicionário Python). Dados são perdidos ao reiniciar o processo — isso é esperado e aceito. Testes devem utilizar estado isolado e relógio determinístico, sem aguardar tempo real.
11. Escutar em `0.0.0.0:{PORTA_SERVICO}` no contêiner e publicar a mesma porta no host. Não fixar 5000 ou 8000.
12. Estes Markdown são especificações, sem implementação. Caso sejam acrescentados snippets, cada um terá no máximo 20 linhas;


## Critério de conclusão

Concluir somente quando os UC1–UC8 estiverem implementados, os testes de contrato e borda passarem e a API responder na porta da variante após `docker compose up -d --build`. Informar verificações não executadas; não declarar sucesso sem evidência.

---

## Regra de Content-Type e erros de validação

- Todas as respostas têm `Content-Type: application/json`.
- O servidor deve registrar um handler global para erros de validação automáticos do framework (ex.: `RequestValidationError` no FastAPI / handler de erro 422 no Flask) que retorne o envelope `{"erro": "..."}` correto. Sem esse handler, o framework retorna por padrão `{"detail": [...]}` ou similar, quebrando o contrato.
- Requisições com body devem enviar `Content-Type: application/json`. Outros content-types resultam em erro 422, mas o body da resposta de erro deve seguir o envelope padrão: `{"erro": "placa_invalida"}` ou equivalente.

## Parâmetros da variante (confirmados)

| Parâmetro | Valor |
|---|---|
| `TARIFA_HORA_CENTAVOS` | `400` |
| `FRACAO_MINUTOS` | `30` |
| `TETO_DIARIO_CENTAVOS` | `5000` |
| `TOLERANCIA_MINUTOS` | `15` |
| `PORTA_SERVICO` | `8001` |

`valor_por_fracao = 400 // (60 // 30) = 200` centavos por fração de 30 minutos.