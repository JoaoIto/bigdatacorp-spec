# 🏗️ Architecture Design — Pipeline de Processamento Batch

> **Projeto:** Desafio Técnico BigDataCorp  
> **Data:** 2026-08-09  
> **Status:** Aprovado para implementação

---

## 1. Padrões de Projeto Adotados

### 1.1 Pipes and Filters (Pipeline Pattern)

O padrão **Pipes and Filters** é a espinha dorsal da solução. Cada "filtro" é uma unidade de processamento independente que recebe dados de uma fonte (pipe), transforma-os e os passa adiante. As unidades são compostas sequencialmente.

**Por que este padrão?**

| Propriedade             | Benefício para o desafio                                                   |
|-------------------------|-----------------------------------------------------------------------------|
| Modularidade            | Cada estágio (leitura, validação, transformação, escrita) é isolado        |
| Testabilidade           | Cada filtro pode ser testado unitariamente sem depender dos outros          |
| Streaming natural       | Os dados fluem um registro por vez, sem acumular em memória                |
| Tolerância a falhas     | Um erro em um registro não contamina o pipeline — o próximo registro segue |
| Extensibilidade         | Novos filtros (ex: novo formato de saída) podem ser inseridos sem refatoração |

### 1.2 Generator Pattern (Lazy Evaluation)

Os **generators** do Python (`yield`) são o mecanismo que torna o streaming possível. Um generator produz valores **sob demanda** — ele não materializa toda a sequência em memória.

```
Generator vs Lista:

lista = [process(x) for x in arquivo]   # ❌ Materializa tudo: O(N) de espaço
gen   = (process(x) for x in arquivo)   # ✅ Lazy: O(1) de espaço
```

Na nossa solução, o **Reader** é um generator que faz `yield` de um registro por vez. O **consumidor** (Writer) puxa sob demanda. A memória utilizada é sempre proporcional a **um único registro**, independentemente do tamanho do arquivo.

### 1.3 Fail-Fast no Registro, Fail-Safe no Pipeline

```
Registro individual:  FAIL-FAST  → Detecta problemas o mais cedo possível.
Pipeline como um todo: FAIL-SAFE → Nunca aborta por causa de um registro ruim.
```

Isso é implementado com `try/except` **dentro** do loop de iteração, envolvendo cada registro individualmente. O pipeline continua mesmo que 50% dos registros sejam inválidos.

---

## 2. Fluxo de Dados — Do JSONL ao CSV

### 2.1 Visão Geral do Pipeline

```
                          ┌─────────────────────────────────────────────┐
                          │          PIPELINE DE STREAMING              │
                          │                                             │
  ┌──────────┐            │  ┌────────┐   ┌──────────┐   ┌──────────┐  │   ┌──────────┐
  │          │  readline   │  │        │   │          │   │          │  │   │          │
  │  JSONL   │───────────▶│  │ READER │──▶│TRANSFORM │──▶│ FILTER   │──│──▶│ WRITERS  │
  │  File    │  (1 linha)  │  │        │   │          │   │          │  │   │          │
  │          │            │  │ yield  │   │ map cols │   │ Série    │  │   │clubs.csv │
  │ (milhões │            │  │ record │   │ fmt date │   │ A/B only │  │   │players   │
  │  linhas) │            │  │        │   │ fmt color│   │          │  │   │ .csv     │
  └──────────┘            │  └────────┘   └──────────┘   └──────────┘  │   └──────────┘
                          │      O(1)         O(1)           O(1)      │       O(1)
                          │                                             │
                          │  Memória total: O(1) — 1 registro por vez   │
                          └─────────────────────────────────────────────┘
```

### 2.2 Estágios Detalhados

#### Estágio 1: Reader (Leitura + Parse)

```python
def read_jsonl(filepath):
    """Generator que lê o JSONL linha a linha e faz yield de dicts válidos."""
    with open(filepath, 'r', encoding='utf-8') as f:
        for line_number, raw_line in enumerate(f, start=1):
            stripped = raw_line.strip()
            if not stripped:
                continue               # Linhas vazias são ignoradas
            try:
                record = json.loads(stripped)
            except json.JSONDecodeError as e:
                logger.warning(f"Linha {line_number}: JSON malformado — {e}")
                stats['linhas_invalidas'] += 1
                continue                # Registro descartado, pipeline segue
            if not isinstance(record, dict):
                logger.warning(f"Linha {line_number}: esperado objeto JSON, obteve {type(record).__name__}")
                stats['linhas_invalidas'] += 1
                continue
            yield line_number, record
```

**Características de memória:**
- `for line in f` usa o iterador nativo do Python, que lê com buffer interno (~8KB). **Não carrega o arquivo inteiro.**
- `json.loads(stripped)` parseia uma única string. O dict resultante existe em memória somente até o próximo `yield`.
- O generator é **lazy** — só executa quando o consumidor chama `next()`.

#### Estágio 2: Filter (Filtro de Campeonato)

```python
def filter_by_championship(records):
    """Generator que filtra apenas clubes da Série A ou B."""
    for line_number, record in records:
        if is_valid_championship(record):
            yield line_number, record
        else:
            stats['filtrados_campeonato'] += 1
```

**Por que um estágio separado?**
- Responsabilidade única: o Reader não precisa conhecer regras de negócio.
- Composabilidade: se amanhã o filtro mudar (ex: incluir Série C), apenas este estágio muda.

#### Estágio 3: Transformer (Mapeamento e Formatação)

Não é um generator em si — é um conjunto de funções puras que transformam um `dict` em uma `list` (a row do CSV). São chamadas sob demanda para cada registro:

```
transform_club(record)  → [club_id, name, championship, ...]
transform_player(club_id, player_dict) → [club_id, player_id, name, ...]
```

Funções auxiliares:
- `format_date(value)` — Valida e formata datas.
- `format_colors(colors)` — Une lista com `|`.
- `safe_str(record, key)` — Extrai campo com fallback para `""`.

#### Estágio 4: Writer (Escrita CSV)

Os writers são simples wrappers em torno de `csv.writer`. Eles recebem rows já transformadas e escrevem imediatamente — sem buffer intermediário.

```python
clubs_writer.writerow(transform_club(record))
for player in players:
    players_writer.writerow(transform_player(club_id, player))
```

### 2.3 Composição do Pipeline

A beleza do Generator Pattern é que os estágios se compõem por aninhamento:

```python
# Composição do pipeline:
raw_records = read_jsonl(input_path)                    # Generator 1
filtered    = filter_by_championship(raw_records)       # Generator 2 (consome G1)

for line_number, record in filtered:                    # Consumidor final
    clubs_writer.writerow(transform_club(record))
    for player in safe_get(record, 'players', []):
        players_writer.writerow(transform_player(...))
```

Cada `next()` no pipeline puxa um registro do JSONL, filtra, transforma e escreve. **A memória nunca ultrapassa o tamanho de um único registro + seus jogadores.**

---

## 3. Modelo de Separação de Responsabilidades

### 3.1 Diagrama de Módulos

```
┌─────────────────────────────────────────────────────────────────────┐
│                         main.py                                     │
│                     (Orquestrador)                                  │
│                                                                     │
│  Responsabilidades:                                                 │
│  ✦ Parsing de argumentos CLI (sys.argv)                             │
│  ✦ Composição do pipeline (conecta Reader → Filter → Writer)       │
│  ✦ Gerenciamento do ciclo de vida de arquivos (open/close)          │
│  ✦ Contadores e relatório final de processamento                   │
│  ✦ Ponto de entrada único — importável e executável                │
│                                                                     │
│  Dependências: reader, transformer, writer                          │
└──────┬──────────────────┬──────────────────┬────────────────────────┘
       │                  │                  │
       ▼                  ▼                  ▼
┌──────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  reader.py   │  │ transformer.py   │  │   writer.py      │
│  (I/O In)    │  │ (Regras/Lógica)  │  │   (I/O Out)      │
│              │  │                  │  │                  │
│ ✦ read_jsonl │  │ ✦ is_valid_champ │  │ ✦ create_writers │
│   (generator)│  │ ✦ transform_club │  │ ✦ write_club_row │
│ ✦ Tratamento │  │ ✦ transform_player│ │ ✦ write_player   │
│   de linhas  │  │ ✦ format_date    │  │    _row          │
│   malformadas│  │ ✦ format_colors  │  │ ✦ write_headers  │
│              │  │ ✦ safe_str       │  │                  │
│ Sem regras   │  │ Sem I/O          │  │ Sem regras       │
│ de negócio!  │  │ de arquivos!     │  │ de negócio!      │
└──────────────┘  └──────────────────┘  └──────────────────┘
       │                  │                  │
       └──────────────────┼──────────────────┘
                          │
                    Princípio:
              Cada módulo tem UMA
              e somente UMA razão
                  para mudar.
```

### 3.2 Matriz de Responsabilidades (RACI simplificado)

| Responsabilidade                      | `main.py` | `reader.py` | `transformer.py` | `writer.py` |
|---------------------------------------|:---------:|:-----------:|:-----------------:|:-----------:|
| Parse de argumentos CLI               |     ✦     |             |                   |             |
| Abertura/fechamento de arquivos       |     ✦     |             |                   |             |
| Leitura do JSONL em streaming         |           |      ✦      |                   |             |
| Parse JSON seguro (`try/except`)      |           |      ✦      |                   |             |
| Filtro por campeonato                 |           |             |        ✦          |             |
| Formatação de datas                   |           |             |        ✦          |             |
| Formatação de cores                   |           |             |        ✦          |             |
| Mapeamento JSON → row CSV            |           |             |        ✦          |             |
| Escrita do CSV (RFC 4180)             |           |             |                   |      ✦      |
| Logging de erros/warnings             |     ✦     |      ✦      |                   |             |
| Contadores e relatório final          |     ✦     |             |                   |             |

### 3.3 Fronteiras e Contratos

```
reader.py EXPÕE:
  read_jsonl(filepath) → Generator[Tuple[int, dict]]
  ↳ Produz (line_number, record) para cada JSON válido.
  ↳ Ignora linhas vazias e JSONs malformados (logando warnings).

transformer.py EXPÕE:
  is_valid_championship(record) → bool
  transform_club(record) → List[str]
  transform_player(club_id, player) → List[str]
  format_date(value) → str
  format_colors(colors) → str
  safe_str(record, key) → str
  CLUBS_HEADER → List[str]     (constantes)
  PLAYERS_HEADER → List[str]   (constantes)

writer.py EXPÕE:
  create_club_writer(filepath) → Tuple[file, csv.writer]
  create_player_writer(filepath) → Tuple[file, csv.writer]
  ↳ Abre o arquivo e retorna o writer já com header escrito.
```

---

## 4. Garantia de Complexidade de Espaço O(1)

### 4.1 Análise Formal de Memória

| Variável em memória     | Tamanho                              | Classificação |
|-------------------------|--------------------------------------|:-------------:|
| Buffer de leitura       | ~8KB (buffer interno do Python)      | O(1)          |
| `raw_line` (string)     | Tamanho de 1 linha JSONL             | O(1)*         |
| `record` (dict)         | 1 clube + N jogadores                | O(1)*         |
| `club_row` (list)       | 11 strings                           | O(1)          |
| `player_row` (list)     | 8 strings                            | O(1)          |
| Buffer de escrita CSV   | ~8KB (buffer interno do Python)      | O(1)          |
| Contadores (stats)      | ~5 inteiros                          | O(1)          |
| Logger                  | Referência fixa                      | O(1)          |
| **Total**               | **~16KB + tamanho do maior registro** | **O(1)**      |

> \*O(1) no sentido de que o tamanho é limitado pelo **maior registro individual**, não pelo número total de registros no arquivo. Em termos formais, se $M$ é o tamanho do maior registro e $N$ o número total de registros, a memória é $O(M)$, que é independente de $N$.

### 4.2 Anti-padrões Evitados

| Anti-padrão                                     | Por que é O(N)                            | Nossa abordagem O(1)                     |
|--------------------------------------------------|-------------------------------------------|------------------------------------------|
| `data = json.load(file)`                         | Carrega todo o JSON na RAM                | `json.loads(line)` — uma linha por vez   |
| `lines = file.readlines()`                       | Lista com todas as linhas                 | `for line in file` — iterador lazy       |
| `all_clubs = []; all_clubs.append(row)`          | Acumula todos os resultados               | `writer.writerow(row)` — escrita imediata|
| `df = pd.read_json(file, lines=True)`            | DataFrame materializa tudo                | Sem pandas — streaming puro              |

---

## 5. Tratamento de Erros no Pipeline

### 5.1 Taxonomia de Erros

```
Erros
├── Erros de Infraestrutura (FATAIS — abortam o programa)
│   ├── Arquivo de entrada não encontrado
│   ├── Sem permissão de escrita no diretório de saída
│   └── Disco cheio durante escrita
│
└── Erros de Dados (NÃO-FATAIS — registro ignorado, pipeline continua)
    ├── Erros de Estrutura (linha inteira descartada)
    │   ├── Linha não é JSON válido
    │   └── JSON válido mas não é um dict
    │
    └── Erros de Validação (campo individual → vazio, registro preservado)
        ├── Data com formato inválido → campo vazio
        ├── `colors` não é uma lista → campo vazio
        └── Campo ausente ou null → campo vazio
```

### 5.2 Onde cada erro é tratado

| Camada       | Erro tratado                      | Ação                        |
|--------------|-----------------------------------|-----------------------------|
| `main.py`    | Arquivo não encontrado, I/O       | `sys.exit(1)` com mensagem  |
| `reader.py`  | JSON malformado, tipo inesperado  | `continue` + log warning    |
| `transformer`| Data inválida, colors inválido    | Retorna `""` (campo vazio)  |

---

## 6. Considerações de Performance para I/O Bound

### 6.1 Perfil do Workload

Este pipeline é **I/O bound**, não CPU bound:
- A maior parte do tempo é gasta **lendo** do disco e **escrevendo** no disco.
- O processamento por registro (parse JSON, formatação) é trivial em comparação.

### 6.2 Otimizações Aplicadas

| Técnica                              | Descrição                                                            |
|--------------------------------------|----------------------------------------------------------------------|
| Iterador nativo do arquivo           | `for line in file` usa buffer interno otimizado (~8KB)               |
| Sem `readlines()`                    | Evita materializar todas as linhas em memória                        |
| `csv.writer` com escrita direta      | Sem buffer intermediário de linhas CSV                               |
| `json.loads()` (não `json.load()`)   | Parse de string, não de file object — mais controlável               |
| Funções puras no transformer         | Sem alocações desnecessárias, sem side effects                       |

### 6.3 Por que NÃO usar multiprocessing/async?

Para este caso, a complexidade adicional **não se justifica**:

1. **I/O sequencial:** Lemos um arquivo de entrada e escrevemos dois de saída. Não há paralelismo natural.
2. **Overhead de IPC:** Multiprocessing exige serializar/deserializar dados entre processos.
3. **Complexidade de código:** O enunciado valoriza **clareza e organização**. Async/multiprocessing adicionam complexidade sem benefício mensurável.
4. **Performance suficiente:** Python puro com streaming processa ~500K–1M linhas/segundo. Para milhões de registros, isso significa minutos — totalmente aceitável para batch.

---

## 7. Próxima Fronteira (Roadmap de Escala Corporativa)

O sistema atual atende perfeitamente ao requisito $O(1)$ de memória para processamento local de arquivos JSONL gigantes. No entanto, para escalar para um ecossistema cloud real (Petabytes) e topologias de sistemas distribuídos (SRE/DevOps), o design precisaria evoluir em três eixos principais:

### 7.1 Escala Horizontal via Sharding
A leitura sequencial impõe um teto de vazão limitado à capacidade de um único disco/CPU.
- **Evolução:** Em vez de rodar o script contra um arquivo inteiro, o orquestrador (ex: Airflow) quebraria o arquivo gigante em N shards ou leria diretamente de um barramento de eventos (ex: Amazon SQS, Kafka).
- **Execução:** O container do nosso pipeline se tornaria completamente **stateless**. Centenas de réplicas processariam os shards em paralelo (ex: AWS Fargate, Kubernetes) e fariam o dump dos resultados diretamente num Data Lake (AWS S3) com tabelas versionadas no Athena/Iceberg, em vez de arquivos CSV locais.

### 7.2 Resumabilidade (Checkpointing)
Atualmente, se o processamento abortar na linha 9.999.999, graças à idempotência (escritas em `.tmp`), evitamos corromper a base final. Porém, precisamos reiniciar do zero.
- **Evolução:** Implementar um mecanismo de gravação do `byte-offset` do arquivo de entrada a cada $X$ registros. 
- **Recuperação:** Num reinício, o `reader.py` usaria `file.seek(offset)` para pular instantaneamente todo o arquivo já processado, retomando a execução do exato ponto de falha.

### 7.3 Prevenção de Estouro de Linha Única (Streaming JSON Parser)
Nossa promessa $O(1)$ assegura que a memória só cresce até o tamanho da maior linha $O(M)$. Mas e se um clube mal-intencionado vier com 5 milhões de jogadores numa única string JSONL, totalizando 15GB numa só linha? O buffer da RAM iria estourar.
- **Evolução:** Substituir `json.loads` (que exige a string inteira em RAM) por um parser SAX-like para JSON em C, como o **`ijson`** (ou `yajl`). Ele permite ler arrays JSON emitindo eventos (prefixo/item) de forma verdadeiramente iterativa, nunca carregando a linha inteira em memória.
