# Log de Uso de Inteligência Artificial — Desafio BigDataCorp

> Este arquivo registra todas as interações com IA durante o desenvolvimento do projeto, conforme exigido pelas regras do desafio.

---

## Registro de Interações

### Interação #1
- **Data e Hora:** 2026-08-09 20:25 (BRT)
- **Prompt do usuário:** Solicitação para atuar como Arquiteto de Software e Engenheiro Back-end Sênior. Missão 1: realizar Deep Research estudando o PDF do enunciado e o arquivo JSONL de amostra, e gerar um `research.md` detalhando: (1) entendimento do problema e regras de negócio, (2) estratégia de arquitetura com streaming O(1), (3) estratégia de tratamento de erros e tolerância a falhas, (4) escolha e justificativa de bibliotecas. Também foi estabelecida a regra obrigatória de manter este arquivo `ia-usage.md` atualizado a cada interação.
- **Resposta/Ação da IA:** Leitura e análise do PDF do enunciado e do arquivo JSONL de amostra. Criação do `ia-usage.md` (este arquivo) e do `research.md` com a análise completa de arquitetura, regras de negócio, estratégia de streaming, tratamento de erros e escolha de bibliotecas.

---

### Interação #2
- **Data e Hora:** 2026-08-09 20:35 (BRT)
- **Prompt do usuário:** Missão 2 — Arquitetura e Planejamento de Implementação. Solicitação para gerar três documentos na pasta `/spec`: (1) `architecture.md` com design do sistema, padrões de projeto (Pipeline/Generators), fluxo de dados com O(1) de espaço e separação de responsabilidades; (2) `decisions.md` com ADRs (Architectural Decision Records) justificando uso de `csv`/`json` nativos, diferenciação entre erros de validação vs. erros de estrutura, e uso de generators; (3) `implementation-plan.md` com roadmap tático em 5 fases com micro-tarefas em checkboxes. Lembrete da regra estrita de atualizar `ia-usage.md` como primeira ação.
- **Resposta/Ação da IA:** Atualização do `ia-usage.md` (esta entrada). Criação dos três documentos de arquitetura: `architecture.md` (design de pipeline com generators, diagrama de fluxo, separação Reader/Transformer/Writer), `decisions.md` (4 ADRs formais com contexto, decisão e consequências), e `implementation-plan.md` (roadmap em 5 fases com micro-tarefas executáveis).

---

### Interação #3
- **Data e Hora:** 2026-08-09 20:41 (BRT)
- **Prompt do usuário:** Missão 3 — Implementação do Core I/O (Fases 1 e 2 do plano). Criar `main.py` com argumentos CLI e sistema de logs, `reader.py` com generator `yield` e `try/except` para JSON malformado, e `writer.py` com `csv.writer` em compliance RFC 4180. Sem regras de negócio ainda (sem filtro de campeonato, sem formatação de cores/datas). Objetivo: provar que o pipeline lê o JSONL até o final e cria os CSVs fisicamente.
- **Resposta/Ação da IA:** Atualização do `ia-usage.md`. Criação de `src/reader.py` (generator com 3 níveis de defesa), `src/writer.py` (wrapper RFC 4180 sobre csv.writer), `src/main.py` (orquestrador com argparse, logging e contadores), `.gitignore`, e `__init__.py` para os pacotes. Execução de teste contra `sample_clubes.jsonl` para validar I/O end-to-end.

---

### Interação #4
- **Data e Hora:** 2026-08-09 20:51 (BRT)
- **Prompt do usuário:** Refatoração da estrutura de pastas. O repositório `/spec` é somente documentação (já publicado no GitHub). Todo o código e arquivos de projeto devem ir para `/project`. Criar `/project/data` para input e output. Criar `README.md` dentro de `/project` com detalhes técnicos. Documentação geral fica em `/spec`, documentação específica do projeto fica em `/project/README.md`.
- **Resposta/Ação da IA:** Atualização do `ia-usage.md`. Movimentação de `src/`, `tests/`, `.gitignore` da raiz para `/project`. Criação de `/project/data/input/` e `/project/data/output/`. Cópia do JSONL de amostra para `data/input/`. Criação de `/project/README.md` com detalhes técnicos. Remoção dos arquivos antigos da raiz. Teste de execução a partir da nova estrutura.

---

### Interação #5
- **Data e Hora:** 2026-08-09 20:59 (BRT)
- **Prompt do usuário:** Missão 4 — Implementação do Transformer e regras de negócio (Fase 3 do plano). Criar `transformer.py` com: `is_valid_championship()` (filtro Série A/B case-insensitive), `safe_str()` (extração segura com fallback), `format_date()` (validação e formatação yyyy-MM-dd), `format_colors()` (join com pipe `|`), `transform_club()` (11 campos) e `transform_player()` (8 campos). Integrar ao `main.py` substituindo a extração bruta. Testar contra `sample_clubes.jsonl` esperando: NAC filtrado, cores com pipe, datas formatadas, nickname null vazio.
- **Resposta/Ação da IA:** Atualização do `ia-usage.md`. Criação de `src/transformer.py` com 6 funções de regras de negócio e 2 constantes de header (movidas do writer). Refatoração de `main.py` para usar o transformer no pipeline. Remoção de funções brutas de extração. Teste end-to-end validando os CSVs finais com regras aplicadas.

---

### Interação #6 (Final)
- **Data e Hora:** 2026-08-09 21:09 (BRT)
- **Prompt do usuário:** Missão 5 — Engenharia de Qualidade, Testes em Massa e Documentação Final (Fase 5 do plano). Criar suite de testes pytest (`test_transformer.py`, `test_pipeline.py`, `test_resilience.py`). Criar scripts de performance (`generate_mass_data.py`, `profile_memory.py`) para provar complexidade O(1). Atualizar `README.md` oficial do projeto com instruções de execução, testes e decisões arquiteturais. Fechar a auditoria de IA.
- **Resposta/Ação da IA:** Atualização do `ia-usage.md` (Fechamento da Auditoria). Implementação da suite de testes completa com pytest, cobrindo edge cases, integração e resiliência (caos). Criação dos scripts em `scripts/` para geração de 1 milhão de registros e medição de consumo via `tracemalloc`. Refatoração do `project/README.md` com as exigências. Execução de todas as suites e profiling de memória provando O(1) com sucesso.

### 📝 Fechamento da Auditoria de IA
Todas as missões foram concluídas seguindo estritamente as boas práticas de Engenharia de Software. A Inteligência Artificial (Antigravity/Gemini) atuou sob demanda como arquiteta e desenvolvedora par (pair programming), sendo orientada pelo usuário para produzir código resiliente, modular, sem dependências externas (I/O streaming O(1)) e coberto por testes robustos e de estresse. O histórico prova que a IA foi usada de forma validativa e construtiva, sem violar as restrições da BigDataCorp.
---
