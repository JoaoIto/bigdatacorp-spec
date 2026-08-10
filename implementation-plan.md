# 🗺️ Plano de Implementação — Desafio BigDataCorp

> **Projeto:** Processamento Batch JSONL → CSV  
> **Data:** 2026-08-09  
> **Status:** Em execução — Fases 1 e 2 concluídas  
> **Documentos de referência:** `research.md`, `architecture.md`, `decisions.md`

---

## Fase 1: Setup e Infraestrutura

**Objetivo:** Preparar a estrutura do projeto, configuração de qualidade de código e repositório Git.

- [x] **1.1** Criar estrutura de diretórios do projeto:
  ```
  bigdatacorp/
  ├── spec/                         # Repositório de documentação (GitHub)
  │   ├── docs/                     # PDF do enunciado e JSONL original
  │   ├── research.md
  │   ├── architecture.md
  │   ├── decisions.md
  │   ├── implementation-plan.md
  │   └── ia-usage.md
  └── project/                      # Repositório do código (GitHub)
      ├── src/
      │   ├── main.py
      │   ├── reader.py
      │   ├── transformer.py        (Fase 3)
      │   └── writer.py
      ├── tests/
      ├── data/
      │   ├── input/                # Arquivos JSONL de entrada
      │   └── output/               # CSVs gerados (gitignored)
      ├── .gitignore
      └── README.md
  ```
- [x] **1.2** Criar `.gitignore` com exclusões para Python (`__pycache__/`, `*.pyc`, `.venv/`, `data/output/`).
- [x] **1.3** Criar `README.md` em `/project` com instruções de uso e detalhes técnicos.
- [ ] **1.4** Inicializar repositório Git com commit inicial de infraestrutura.

**Critério de conclusão:** Estrutura de pastas criada, repositório Git inicializado.

---

## Fase 2: Core de I/O (Streaming)

**Objetivo:** Implementar as camadas de leitura (Reader) e escrita (Writer) com streaming O(1).

### 2A — Reader (Leitura JSONL)

- [x] **2.1** Implementar `reader.py` com o generator `read_jsonl(filepath)`:
  - Assinatura: `read_jsonl(filepath: str) → Generator[Tuple[int, dict], None, None]`
  - Itera sobre o arquivo linha a linha com `for line in file`.
  - Para cada linha:
    - Ignora linhas vazias (whitespace-only).
    - Tenta `json.loads(stripped)`.
    - Se falhar (`JSONDecodeError`), loga WARNING e incrementa contador — `continue`.
    - Se o resultado não for `dict`, loga WARNING — `continue`.
    - Se sucesso, faz `yield (line_number, record)`.
- [x] **2.2** Garantir que o `open()` usa `encoding='utf-8'`.

### 2B — Writer (Escrita CSV)

- [x] **2.3** Implementar `writer.py` com funções de criação dos writers:
  - `create_csv_writer(filepath, header)` → abre arquivo com `newline=''` e `encoding='utf-8'`, cria `csv.writer`, escreve o header, retorna `(file_handle, csv_writer)`.
- [x] **2.4** Garantir que o `open()` usa `newline=''` (obrigatório para o módulo `csv` no Windows — evita `\r\r\n`).
- [x] **2.5** Definir as constantes de cabeçalho em `writer.py` (movidas para writer por coesão nesta fase):
  - `CLUBS_HEADER`: Lista com os 11 nomes de coluna em português, na ordem exata.
  - `PLAYERS_HEADER`: Lista com os 8 nomes de coluna em português, na ordem exata.

**Critério de conclusão:** Reader lê o sample sem erros. Writer cria CSVs com headers corretos.

---

## Fase 3: Regras de Negócio e Parsers

**Objetivo:** Implementar todas as regras de negócio documentadas em `research.md` (RN01–RN08).

### 3A — Filtro de Campeonato (RN01)

- [x] **3.1** Implementar `is_valid_championship(record) → bool` em `transformer.py`:
  - Extrai `record.get('championship')`.
  - Se `None` ou não-string → retorna `False`.
  - Normaliza com `.strip().upper()`.
  - Verifica se está em `{'SERIE A', 'SERIE B'}`.

### 3B — Extração Segura de Campos (RN05)

- [x] **3.2** Implementar `safe_str(record, key) → str` em `transformer.py`:
  - `record.get(key)` — se `None` ou ausente, retorna `""`.
  - Caso contrário, retorna `str(value)`.
  - Trata o caso especial: valores numéricos (ex: `age=26`) devem virar `"26"`, não falhar.

### 3C — Formatação de Datas (RN04)

- [x] **3.3** Implementar `format_date(value) → str` em `transformer.py`:
  - Se `value` é `None`, não é string, ou é string vazia → retorna `""`.
  - Tenta parsear com `datetime.strptime()` nos formatos: `'%Y-%m-%d'`, `'%d/%m/%Y'`, `'%Y/%m/%d'`.
  - Se algum suceder → retorna `date.strftime('%Y-%m-%d')`.
  - Se todos falharem → retorna `""` (campo vazio, registro preservado).

### 3D — Formatação de Cores (RN03)

- [x] **3.4** Implementar `format_colors(colors) → str` em `transformer.py`:
  - Se `colors` é `None` ou não é `list` → retorna `""`.
  - Filtra elementos `None` da lista.
  - Une com `'|'.join(...)`.
  - Lista vazia após filtragem → retorna `""`.

### 3E — Funções de Transformação (Mapeamento JSON → CSV Row)

- [x] **3.5** Implementar `transform_club(record) → list` em `transformer.py`:
  - Extrai os 11 campos do clube na ordem exata do CSV, usando `safe_str`, `format_date`, `format_colors`.
  - Campos extraídos: `club_id`, `name`, `championship`, `founding_date`, `city`, `state`, `country`, `stadium`, `president`, `nickname`, `colors`.
  - Campos descartados (não mapeados): `titles`, `players`.
  
- [x] **3.6** Implementar `transform_player(club_id, player) → list` em `transformer.py`:
  - Extrai os 8 campos do jogador na ordem exata do CSV, usando `safe_str`, `format_date`.
  - Primeiro campo é o `club_id` do clube pai (passado como argumento).
  - Campos descartados: `nationality`, `market_value`.

**Critério de conclusão:** Todas as funções de transformação implementadas e cobrindo as regras RN01–RN05.

---

## Fase 4: Orquestração e Tolerância a Falhas

**Objetivo:** Integrar todas as camadas no `main.py` com o pipeline completo e logging robusto.

### 4A — Orquestrador Principal

- [x] **4.1** Implementar `main.py` com a função `process(input_path, output_dir)`:
  - Compõe o pipeline: `read_jsonl()` → filtragem → transformação → escrita.
  - Abre os 3 arquivos (1 input + 2 output) com gerenciamento de contexto (`with`).
  - Itera sobre os registros válidos e filtrados.
  - Para cada registro, escreve em `clubs.csv` e itera sobre `players` para escrever em `players.csv`.

- [x] **4.2** Implementar tratamento de argumentos CLI:
  - `sys.argv[1]` — caminho do arquivo de entrada (obrigatório).
  - `sys.argv[2]` — diretório de saída (opcional; padrão = diretório do arquivo de entrada).
  - Validar que o arquivo de entrada existe (`os.path.isfile()`).
  - Validar que o diretório de saída existe ou criá-lo (`os.makedirs(exist_ok=True)`).

### 4B — Tolerância a Falhas

- [x] **4.3** Envolver o processamento de cada registro em `try/except Exception`:
  - Erros inesperados em um registro individual são logados como WARNING.
  - O pipeline segue para o próximo registro.
  - Apenas erros de infraestrutura (arquivo não encontrado, I/O) abortam.

- [x] **4.4** Implementar contadores de processamento:
  - `linhas_lidas` — total de linhas no arquivo.
  - `linhas_vazias` — linhas em branco ignoradas.
  - `linhas_json_invalido` — JSONs malformados.
  - `linhas_filtradas` — campeonato fora do filtro.
  - `clubes_escritos` — registros em clubs.csv.
  - `jogadores_escritos` — registros em players.csv.

### 4C — Logging e Relatório Final

- [x] **4.5** Configurar `logging` com formato padronizado:
  - Formato: `%(asctime)s [%(levelname)s] %(message)s`
  - Nível padrão: `INFO`.

- [x] **4.6** Implementar relatório final no console ao término:
  ```
  ✅ Processamento concluído com sucesso.
     Linhas lidas:              1.000.000
     Linhas ignoradas (JSON):          42
     Clubes filtrados:            150.000
     Clubes escritos:             849.958
     Jogadores escritos:       12.500.000
  ```

**Critério de conclusão:** `python main.py sample_clubes.jsonl` gera os dois CSVs corretamente a partir da amostra.

---

## Fase 5: Testes, Validação e Documentação Final

**Objetivo:** Garantir corretude com testes unitários, validar a saída contra os resultados esperados, e finalizar a documentação.

### 5A — Testes Unitários

- [ ] **5.1** Implementar `tests/test_transformer.py`:
  - Testar `is_valid_championship()`: SERIE A ✅, SERIE B ✅, SEM CAMPEONATO ❌, None ❌, vazio ❌.
  - Testar `format_date()`: data válida, data inválida, None, string vazia, formato alternativo.
  - Testar `format_colors()`: lista normal, lista vazia, None, elemento None na lista.
  - Testar `safe_str()`: valor presente, valor None, chave ausente, valor numérico.
  - Testar `transform_club()`: registro completo, registro com campos nulos.
  - Testar `transform_player()`: jogador completo, jogador com campos nulos.

- [ ] **5.2** Implementar `tests/test_reader.py`:
  - Testar `read_jsonl()` com arquivo contendo: linhas válidas, linhas vazias, JSON malformado, JSON não-dict.
  - Verificar que linhas inválidas são ignoradas e válidas são yielded corretamente.

- [ ] **5.3** Implementar `tests/test_writer.py`:
  - Testar que o CSV gerado tem encoding UTF-8.
  - Testar que campos com vírgula são escapados corretamente (RFC 4180).
  - Testar que o header está correto.

### 5B — Validação de Integração

- [ ] **5.4** Rodar o programa contra `sample_clubes.jsonl` e verificar:
  - `clubs.csv` contém exatamente **5 clubes** (SCCP, SEP, SFC, CRU, AVA).
  - `players.csv` contém exatamente **8 jogadores** (3+2+2+1+0).
  - O clube NAC (`SEM CAMPEONATO`) **não aparece** em nenhum dos dois arquivos.
  - O clube AVA (SERIE B, sem jogadores) **aparece** em `clubs.csv` mas **não gera linhas** em `players.csv`.
  - O campo `nickname` do SFC é vazio (era `null` no JSON).
  - O campo `president` do CRU (`"Pedro Lourenço, Filho"`) está corretamente entre aspas no CSV.
  - O campo `Cores` de todos os clubes usa `|` como separador.

- [ ] **5.5** Criar arquivo de teste com dados sujos (JSON malformado, campos extras, tipos errados) e validar que o programa não aborta.

### 5C — Documentação Final

- [ ] **5.6** Finalizar `README.md` com:
  - Descrição do projeto.
  - Pré-requisitos (Python 3.8+).
  - Instruções de uso: `python src/main.py <caminho_arquivo_entrada> [diretório_saída]`.
  - Instruções para rodar os testes.
  - Estrutura do projeto.
  - Decisões técnicas (link para `spec/decisions.md`).

- [ ] **5.7** Atualizar `ia-usage.md` com todas as interações finais.

- [ ] **5.8** Revisar todos os arquivos de documentação para consistência.

- [ ] **5.9** Commit final e push para repositório GitHub público.

**Critério de conclusão:** Todos os testes passam, CSVs gerados estão corretos, README documentado, repositório pronto para entrega.

---

## Resumo do Roadmap

| Fase | Nome                               | Entregas                                     | Dependência |
|------|-------------------------------------|----------------------------------------------|-------------|
| 1    | Setup e Infraestrutura             | Estrutura de pastas, Git, .gitignore          | Nenhuma     |
| 2    | Core de I/O (Streaming)            | `reader.py`, `writer.py`                      | Fase 1      |
| 3    | Regras de Negócio e Parsers        | `transformer.py` completo                     | Fase 2      |
| 4    | Orquestração e Tolerância a Falhas | `main.py` integrado, logging, contadores      | Fases 2 + 3 |
| 5    | Testes, Validação e Documentação   | Testes, CSVs validados, README, entrega final | Fase 4      |
| 6    | Elevação para Nível Sênior (SRE)   | Type hints, Logs JSON, utf-8-sig, CI/CD, Docker | Fase 5      |

```
Fase 1 ──▶ Fase 2 ──▶ Fase 3 ──┐
                                 ├──▶ Fase 4 ──▶ Fase 5 ──▶ Fase 6
                   (Fase 2) ────┘
```

---

## Fase 6: Elevação para Nível Sênior (Cloud & SRE)

**Objetivo:** Refatorar o código resolvendo 5 gargalos (Pontos Cegos) apontados em Code Review simulado, tornando o projeto "Cloud-Ready".

### 6A — Blindagem de Tipagem
- [x] **6.1** Adicionar type hints explícitos (módulo `typing`) nas assinaturas de `reader.py`.
- [x] **6.2** Adicionar type hints explícitos (módulo `typing`) nas assinaturas de `transformer.py`.
- [x] **6.3** Adicionar type hints e `Dict`/`Any` (se necessário) no `main.py`.

### 6B — Observabilidade Cloud (Logs JSON)
- [x] **6.4** Substituir logs em texto puro por formatação JSON em `src/main.py`.
- [x] **6.5** Implementar `logging.Formatter` customizado nativo para serializar mensagens em JSON.

### 6C — Resiliência a Encodings Window
- [x] **6.6** Atualizar `open()` no `reader.py` para usar `encoding='utf-8-sig'` (tratamento de BOM).
- [x] **6.7** Atualizar `open()` no `reader.py` para usar `errors='replace'` para prevenir corrupção isolada de bytes.

### 6D — Automação e Empacotamento
- [x] **6.8** CI/CD: Criar pipeline GitHub Actions (`.github/workflows/main.yml`) com linting (flake8/mypy) e testes (pytest).
- [x] **6.9** Docker: Criar `Dockerfile` enxuto com multi-stage build ou imagem distroless-like baseada em alpine.
