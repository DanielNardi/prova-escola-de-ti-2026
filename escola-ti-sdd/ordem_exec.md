# ordem exec — Zona Azul Digital

## 1. Papel e objetivo

Você é o agente responsável por transformar as especificações desta pasta em uma API REST funcional de Zona Azul Digital, usando Python + Flask. Conduza a leitura, a implementação, os testes e a entrega do Dockerfile e do Docker Compose.

Este é o ponto de entrada da execução. Os Markdown contêm instruções; não são scripts a serem executados pelo terminal. Leia-os e realize as ações descritas no repositório da aplicação. Não encerre o trabalho apenas apresentando um plano quando houver condições de implementar.

Trabalhe sequencialmente como um único agente. Não dependa de outros agentes, de um harness ou de uma ferramenta específica de orquestração.

## 2. Arquivos e ordem obrigatória de leitura

Localize os documentos em relação à pasta deste arquivo. Leia todos integralmente antes de alterar código.

| Ordem | Documento | O que deve orientar |
| --- | --- | --- |
| 1 | [constitution.md](constitution.md) | Convenções, limites de escopo e invariantes |
| 2 | [specs.md](specs.md) | Contrato HTTP, UC1–UC8, regras e critérios de aceite |
| 3 | [plan.md](plan.md) | Arquitetura, persistência, relógio, variante e contêiner |
| 4 | [tests.md](tests.md) | Cenários obrigatórios, bordas e resultados esperados |
| 5 | [tasks.md](tasks.md) | Tarefas T1–T9, dependências e critérios de conclusão |

O arquivo de especificação neste pacote chama-se `specs.md`.

Verificar também o enunciado e o contrato oficial quando estiverem disponíveis no repositório. Não inventar arquivos ausentes nem considerar que um arquivo foi lido apenas por conhecer seu nome.

## 3. Precedência e conflitos

Respeitar as instruções do ambiente e as orientações explícitas do usuário. Dentro dos documentos do projeto:

- O contrato normativo da prova prevalece sobre exemplos ilustrativos contraditórios e decisões complementares locais.
- `constitution.md` define invariantes; `specs.md` detalha o comportamento público; `plan.md` define como implementá-lo.
- `tests.md` verifica o comportamento especificado. Não alterar requisitos ou resultados esperados apenas para fazer um teste passar.
- `tasks.md` organiza o trabalho; este orquestrador determina a sequência e os pontos de verificação, sem criar novas regras de negócio.

Se encontrar conflito, identificar os trechos e aplicar a fonte de maior precedência. Se a divergência continuar sem solução, bloquear somente o comportamento afetado, registrar o motivo e prosseguir com trabalho independente. Solicitar esclarecimento específico quando necessário.

## 4. Preparação e controle da execução

Antes de implementar:

1. Confirmar a raiz do repositório de destino, os arquivos existentes e eventuais instruções locais aplicáveis.
2. Ler os cinco documentos na ordem da seção 2 e identificar as tarefas já realizadas.
3. Inspecionar a implementação existente antes de criar ou substituir arquivos. Preservar alterações do usuário e dados persistidos.
4. Registrar em `tasks.md` o estado de cada tarefa: `pendente`, `em andamento`, `bloqueada` ou `concluída`. Manter no máximo uma em andamento.
5. Para cada conclusão, registrar de forma breve a evidência: arquivos produzidos, verificação executada e resultado. Marcar checkboxes apenas após satisfazer o aceite correspondente.

Não criar novos arquivos de especificação para duplicar conteúdos existentes. O orquestrador é um documento auxiliar; não presumir que ele aumenta a pontuação além do limite de cinco arquivos do critério E.

## 5. Sequência de implementação e pontos de verificação

Executar T1 → T2 → T3 → T4 → T5 → T6 → T7 → T8, com a exceção de variante ausente descrita na seção 6.

| Etapa | Referências principais | Trabalho | Condição para avançar |
| --- | --- | --- | --- |
| T1 — Variante | constitution.md | Variante já confirmada em `variante/params.json` — carregar valores em `app/config.py` | `app/config.py` criado com os cinco parâmetros corretos |
| T2 — Estrutura | plan.md e constitution.md | Criar factory Flask (`criar_app`), `config.py` com constantes da variante, blueprints registrados e `storage.py` em memória | Instância Flask inicia; `GET /bilhetes/ativos` retorna 200 com `[]` |
| T3 — Domínio | specs.md e tests.md | Implementar `calculos.py` e `validacoes.py`: tolerância, frações, teto e arredondamento | Todos os casos da tabela de verificação da Tarefa 2 em `tasks.md` passam |
| T4 — Endpoints bilhetes | specs.md e tests.md | Implementar blueprint `/bilhetes` (UC1, UC2, UC3, UC5, UC6, UC8) | Critérios de aceite das UCs 1–3, 5–6 e 8 em `specs.md` passam |
| T5 — Relatório | specs.md e tests.md | Implementar blueprint `/relatorios` (UC4) | Critérios da UC4 passam, incluindo half-up e exclusão de cancelados |
| T6 — Suíte | tests.md | Consolidar todos os cenários de borda das 10 regras em `tests.md`; executar com a variante real | Suíte aplicável passa; verificações bloqueadas ficam explicitamente pendentes |
| T7 — Docker | plan.md | Criar `Dockerfile` e `compose.yaml`; construir e subir o serviço | `docker compose up -d --build` funciona; API responde em `localhost:8001` |
| T8 — Revisão | Todos | Conferir rastreabilidade dos oito UCs, tipos inteiros, envelope de erro e porta | Todos os critérios de `constitution.md` marcados como verificados |

Os testes de cada etapa devem ser escritos e executados durante a implementação. T7 consolida a suíte; não é o primeiro momento de testar. Testes de T4 podem usar diretamente serviços/repositórios antes da existência das rotas.

## 6. Variante

A variante está confirmada em `variante/params.json`: `TARIFA_HORA_CENTAVOS=400`, `FRACAO_MINUTOS=30`, `TETO_DIARIO_CENTAVOS=5000`, `TOLERANCIA_MINUTOS=15`, `PORTA_SERVICO=8001`. A etapa T1 está concluída — usar esses valores diretamente em `app/config.py`.

Conferir os cinco nomes exatos ao ler `variante/params.json`. `PORTA_API` não substitui `PORTA_SERVICO`.

## 7. Ciclo de execução de cada tarefa

1. Reler as seções diretamente relacionadas à tarefa e verificar as dependências.
2. Implementar somente o escopo da tarefa atual — não antecipar código de tarefas futuras.
3. Executar as verificações descritas no critério de conclusão da tarefa.
4. Registrar evidência sucinta em `tasks.md` e marcar a tarefa como concluída.
5. Avançar para a próxima tarefa.

### Mapa de dependências entre arquivos gerados

```
config.py                   (sem imports internos — raiz)
    ├── calculos.py          (importa config)
    ├── validacoes.py        (importa config)
    └── models.py            (importa config para constantes de validação)
            └── storage.py   (importa models)
                    ├── routes/bilhetes.py    (importa storage + calculos + validacoes)
                    └── routes/relatorios.py  (importa storage + calculos)
                                └── app/__init__.py  (factory Flask — ponto de entrada)
```

Nenhum módulo deve importar de um módulo que depende dele (sem imports circulares). Em especial: `storage.py` não importa `routes`; `calculos.py` não importa `storage`.

## 8. Verificação Docker

1. Comparar os cinco parâmetros do ambiente efetivo com `variante/params.json`, incluindo possíveis sobrescritas por variáveis do shell.
2. Validar a configuração com `docker compose config`.
3. Construir e iniciar com `docker compose up -d --build`.
4. Consultar `GET /bilhetes/ativos` em `http://localhost:8001` com tentativas limitadas de inicialização e exigir HTTP 200.
5. Verificar abertura, conflito por placa ocupada, encerramento, reabertura, cancelamento, histórico e relatório usando dados de teste isolados.
6. A persistência é em memória — dados são perdidos ao recriar o contêiner, o que é esperado. Não há teste de durabilidade entre reinícios.
7. Registrar comandos, resultados e falhas relevantes; não expor conteúdo sensível do ambiente nos registros.

## 9. Retomada após interrupção

Ler novamente este arquivo e o estado registrado em `tasks.md`. Conferir quais artefatos existem e se as evidências ainda correspondem à versão atual do código. Retomar a primeira tarefa incompleta cujas dependências estejam satisfeitas. Não recriar a aplicação, apagar o banco ou repetir etapas concluídas sem necessidade concreta.

## 10. Critério de encerramento e resposta final

Somente declarar a aplicação concluída quando UC1–UC8, erros contratuais, regras monetárias, variante real, persistência e execução em Docker estiverem verificados conforme os documentos.

Na resposta final da execução, informar:

- O que foi implementado e quais artefatos foram gerados.
- Quais testes foram executados e seus resultados.
- Como configurar a variante e iniciar a aplicação.
- A URL e a porta realmente utilizadas, se a aplicação tiver sido iniciada.
- Qualquer bloqueio ou verificação não executada, com o insumo necessário para concluir.

Manter a restrição de snippets: no máximo 20 linhas por bloco e o limite agregado definido em `constitution.md`. Este orquestrador não contém implementação da API.