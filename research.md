# 🔬 Deep Research — Desafio Técnico BigDataCorp (Batch)

> **Autor:** Assistido por IA (ver `ia-usage.md`)  
> **Data:** 2026-08-09  
> **Versão:** 1.0

---

## 1. Entendimento do Problema e Regras de Negócio

### 1.1 Visão Geral

O desafio consiste em construir um processador batch que:

1. **Lê** um arquivo JSONL (`sample_clubes.jsonl`) onde cada linha é um objeto JSON representando um clube de futebol com seus jogadores.
2. **Transforma** os dados aplicando regras de negócio (filtros, formatação, seleção de colunas).
3. **Gera** dois arquivos CSV relacionais: `clubs.csv` (1:1) e `players.csv` (1:N).

### 1.2 Estrutura de Entrada (JSONL)

Cada linha do JSONL contém um objeto JSON com a seguinte estrutura:

```json
{
  "club_id": "SCCP",
  "name": "Sport Club Corinthians Paulista",
  "championship": "SERIE A",
  "founding_date": "1910-09-01",
  "city": "São Paulo",
  "state": "SP",
  "country": "Brasil",
  "stadium": "Neo Química Arena",
  "president": "Augusto Melo",
  "nickname": "Timão",
  "colors": ["preto", "branco"],
  "titles": 30,                       // ← NÃO vai para o CSV
  "players": [
    {
      "player_id": "SCCP-10",
      "name": "Rodrigo Garro",
      "age": 26,
      "goals": 8,
      "debut_date": "2024-01-18",
      "position": "Meia",
      "shirt_number": 10,
      "nationality": "Argentina",     // ← NÃO vai para o CSV
      "market_value": 12000000        // ← NÃO vai para o CSV
    }
  ]
}
```

> **Observação crítica:** O JSON possui mais campos do que os exigidos nos CSVs. Campos como `titles`, `nationality` e `market_value` devem ser **descartados** silenciosamente.

### 1.3 Estrutura de Saída

#### `clubs.csv` — Relação 1:1 (um registro por clube)

| # | Coluna (CSV)       | Origem no JSON  | Observação                                        |
|---|---------------------|-----------------|---------------------------------------------------|
| 1 | Id do Clube         | `club_id`       |                                                   |
| 2 | Nome                | `name`          |                                                   |
| 3 | Campeonato          | `championship`  |                                                   |
| 4 | Data de Fundação    | `founding_date` | Formato `yyyy-MM-dd`. Data inválida → campo vazio |
| 5 | Cidade              | `city`          |                                                   |
| 6 | Estado              | `state`         |                                                   |
| 7 | País                | `country`       |                                                   |
| 8 | Estádio             | `stadium`       |                                                   |
| 9 | Presidente          | `president`     |                                                   |
| 10| Apelido             | `nickname`      | Pode ser `null` ou ausente → campo vazio          |
| 11| Cores               | `colors`        | Lista unida por `\|`. Vazia/ausente → campo vazio |

#### `players.csv` — Relação 1:N (um registro por jogador)

| # | Coluna (CSV)       | Origem no JSON              | Observação                               |
|---|---------------------|-----------------------------|------------------------------------------|
| 1 | Id do Clube         | `club_id` (do clube pai)    | Chave de ligação com `clubs.csv`         |
| 2 | Id do Jogador       | `players[].player_id`       |                                          |
| 3 | Nome                | `players[].name`            |                                          |
| 4 | Idade               | `players[].age`             |                                          |
| 5 | Gols                | `players[].goals`           |                                          |
| 6 | Data de Estreia     | `players[].debut_date`      | Formato `yyyy-MM-dd`. Inválida → vazio   |
| 7 | Posição             | `players[].position`        |                                          |
| 8 | Número da Camisa    | `players[].shirt_number`    |                                          |

### 1.4 Regras de Negócio — Inventário Completo

#### RN01 — Filtro por Campeonato
- **Regra:** Processar **somente** clubes cujo campo `championship` seja `"SERIE A"` ou `"SERIE B"`.
- **Consequência:** Clubes de qualquer outro campeonato (ex: `"SEM CAMPEONATO"`) são **inteiramente descartados** — não entram em `clubs.csv` nem em `players.csv`.
- **Decisão de design:** A comparação será **case-insensitive** e com `strip()`, para tolerar variações como `"Serie A"`, `" SERIE A "`, `"serie a"`, etc.

#### RN02 — Ligação 1:N (Clube → Jogadores)
- Cada linha de `players.csv` carrega o `club_id` do clube ao qual o jogador pertence.
- Um clube **sem jogadores** (lista `players` vazia ou ausente) **continua aparecendo** em `clubs.csv` (se passar no filtro RN01), mas **não gera nenhuma linha** em `players.csv`.

#### RN03 — Formatação de Cores (`colors`)
- A lista de cores deve ser unida em um único campo, separada por `|` (pipe).
- Exemplo: `["preto", "branco"]` → `preto|branco`
- Lista vazia `[]`, ausente ou `null` → campo vazio `""`.

#### RN04 — Formatação de Datas (`founding_date`, `debut_date`)
- Formato de saída: `yyyy-MM-dd` (ex: `2024-01-18`).
- Se o valor de origem **não for uma data válida** (string malformada, `null`, ausente, número, etc.), o campo fica **vazio** — a linha continua no arquivo normalmente.
- **Estratégia de validação:** Usar `datetime.strptime` com `try/except`. Aceitar pelo menos os formatos: `%Y-%m-%d`, `%d/%m/%Y`, `%Y/%m/%d`. Se nenhum funcionar, campo vazio.

#### RN05 — Campos Ausentes ou Nulos
- Campos ausentes ou `null` no JSON viram campo vazio `""` no CSV.
- Isso vale para **todos** os campos, incluindo `nickname`, `colors`, etc.

#### RN06 — Formato CSV (RFC 4180)
- Arquivos em **UTF-8** (com BOM? Não — o padrão é sem BOM para compatibilidade máxima).
- Linha de cabeçalho presente.
- Separador: **vírgula** (`,`).
- Campos que contenham **vírgula**, **aspas duplas** ou **quebra de linha** devem ser escapados:
  - Campo envolto em aspas duplas.
  - Aspas internas são duplicadas (`"` → `""`).
- O módulo `csv` do Python já implementa RFC 4180 nativamente.

#### RN07 — Robustez contra Dados Sujos
- Registros malformados ou incompletos **não devem abortar** o programa.
- Registros inválidos ficam de fora do resultado e o processamento segue normalmente.
- **O que é "inválido"?** Uma linha que não é um JSON parseável.

#### RN08 — Volume de Dados
- O arquivo real pode ter **muitos milhões de registros**.
- O programa deve processar em **streaming**, sem carregar tudo em memória.

### 1.5 Análise da Amostra (`sample_clubes.jsonl`)

| Linha | `club_id` | `championship`     | Jogadores | **Entra no CSV?** | Observações                           |
|-------|-----------|--------------------|-----------|--------------------|---------------------------------------|
| 1     | SCCP      | SERIE A            | 3         | ✅ SIM             | Caso base completo                    |
| 2     | SEP       | SERIE A            | 2         | ✅ SIM             |                                       |
| 3     | SFC       | SERIE B            | 2         | ✅ SIM             | `nickname: null` → testa RN05        |
| 4     | CRU       | SERIE A            | 1         | ✅ SIM             | `president` tem vírgula → testa RN06 |
| 5     | AVA       | SERIE B            | 0         | ✅ SIM (só clubs)  | Lista vazia → testa RN02             |
| 6     | NAC       | SEM CAMPEONATO     | 0         | ❌ NÃO             | Filtrado pela RN01                    |
| 7     | (vazia)   | —                  | —         | ❌ IGNORADA        | Linha em branco                       |

**Resultados esperados da amostra:**
- `clubs.csv`: **5 registros** (SCCP, SEP, SFC, CRU, AVA) + 1 header = 6 linhas
- `players.csv`: **8 registros** (3+2+2+1+0) + 1 header = 9 linhas

---

## 2. Estratégia de Arquitetura

### 2.1 Princípio Fundamental: Streaming com Complexidade de Espaço O(1)

O arquivo real pode ter milhões de registros. **Não podemos** carregar tudo em memória (ex: `json.load()` em todo o arquivo, listas com todos os registros, etc.).

A estratégia é um **pipeline de streaming linha a linha**:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   JSONL      │────▶│   Parser     │────▶│  Transformer │────▶│  CSV Writer  │
│   (Arquivo)  │     │  (1 linha)   │     │  (Regras)    │     │  (Flush)     │
│              │     │              │     │              │     │              │
│  Leitura     │     │  json.loads  │     │  Filtro      │     │  clubs.csv   │
│  Linha a     │     │  + validação │     │  Formatação  │     │  players.csv │
│  linha       │     │              │     │  Seleção     │     │              │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
     O(1)                 O(1)                 O(1)                 O(1)
```

**Por quê O(1)?**
- Lemos **uma linha** do JSONL por vez (`for line in file`), o Python itera com buffer interno.
- Parseamos **um** JSON por vez (`json.loads(line)`).
- Escrevemos imediatamente nos CSVs. Não acumulamos resultados em memória.
- A memória usada é proporcional ao tamanho de **uma linha** (um clube + seus jogadores), não do arquivo inteiro.

### 2.2 Arquitetura em Camadas

O código será organizado em **3 camadas** com responsabilidades bem definidas:

```
┌─────────────────────────────────────────────┐
│            main.py (Orquestrador)           │
│  - Parse de argumentos CLI (sys.argv)       │
│  - Coordena o pipeline de streaming         │
│  - Gerencia abertura/fechamento de arquivos │
│  - Logging e relatório final                │
└─────────────┬───────────────────────────────┘
              │ usa
┌─────────────▼───────────────────────────────┐
│          transformer.py (Regras)            │
│  - Filtro por campeonato (RN01)             │
│  - Formatação de datas (RN04)              │
│  - Formatação de cores (RN03)              │
│  - Extração/mapeamento de colunas          │
│  - Tratamento de campos nulos (RN05)       │
└─────────────┬───────────────────────────────┘
              │ usa
┌─────────────▼───────────────────────────────┐
│          validator.py (Validação)           │
│  - Parse seguro de JSON (try/except)        │
│  - Validação de campos obrigatórios        │
│  - Validação de datas                       │
│  - Logging de erros (sem abortar)          │
└─────────────────────────────────────────────┘
```

### 2.3 Fluxo de Processamento (Pseudocódigo)

```python
def process(input_path, clubs_path, players_path):
    with open(input_path, 'r', encoding='utf-8') as jsonl_in, \
         open(clubs_path, 'w', newline='', encoding='utf-8') as clubs_out, \
         open(players_path, 'w', newline='', encoding='utf-8') as players_out:

        clubs_writer  = csv.writer(clubs_out)
        players_writer = csv.writer(players_out)

        clubs_writer.writerow(CLUBS_HEADER)
        players_writer.writerow(PLAYERS_HEADER)

        for line_number, raw_line in enumerate(jsonl_in, start=1):
            # 1. Ignorar linhas vazias
            stripped = raw_line.strip()
            if not stripped:
                continue

            # 2. Parse seguro do JSON
            try:
                record = json.loads(stripped)
            except json.JSONDecodeError:
                log_warning(f"Linha {line_number}: JSON malformado, ignorada")
                continue

            # 3. Filtro por campeonato (RN01)
            if not is_valid_championship(record):
                continue

            # 4. Transformar e escrever clube (1:1)
            club_row = transform_club(record)
            clubs_writer.writerow(club_row)

            # 5. Transformar e escrever jogadores (1:N)
            club_id = safe_get(record, 'club_id', '')
            for player in safe_get(record, 'players', []):
                player_row = transform_player(club_id, player)
                players_writer.writerow(player_row)
```

### 2.4 Decisões Arquiteturais

| Decisão | Escolha | Justificativa |
|---------|---------|---------------|
| Encoding de saída | UTF-8 **sem BOM** | RFC 4180 não exige BOM; BOM causa problemas em muitas ferramentas |
| Newline no `open()` | `newline=''` | Obrigatório para o módulo `csv` do Python — evita `\r\r\n` no Windows |
| Estrutura de arquivos | 3 módulos separados | Separação de responsabilidades, testabilidade, manutenibilidade |
| Caminho do arquivo de entrada | Via `sys.argv[1]` | Requisito explícito do enunciado: "recebendo o caminho por parâmetro" |
| Diretório de saída | Mesmo diretório do arquivo de entrada | Conveniência; pode ser parametrizado futuramente |

---

## 3. Estratégia de Tratamento de Erros e Tolerância a Falhas

### 3.1 Princípio: "Fail Gracefully, Log Everything"

O programa **nunca aborta** por causa de um registro problemático. A filosofia é:

> **Registros ruins são ignorados. O programa segue. Tudo é logado.**

### 3.2 Camadas de Defesa

```
Nível 1: Linhas vazias / whitespace-only
  → Detectadas com strip(). Ignoradas silenciosamente.

Nível 2: JSON malformado
  → try/except json.JSONDecodeError. Logado e ignorado.

Nível 3: JSON válido, mas estrutura inesperada (não é dict)
  → isinstance(record, dict). Se não for, logado e ignorado.

Nível 4: Campos ausentes ou nulos
  → Função safe_get(dict, key, default) que retorna default
    para chave ausente ou valor None.

Nível 5: Dados com formato inesperado (datas, cores)
  → Formatação de datas: try/except com múltiplos formatos.
    Se nenhum funcionar → campo vazio.
  → Colors: verifica se é lista. Se não for → campo vazio.

Nível 6: Tipos inesperados em campos individuais
  → Conversão via str() com fallback para "".
```

### 3.3 Sistema de Logging

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s'
)
logger = logging.getLogger(__name__)
```

- **INFO:** Início/fim do processamento, contadores finais.
- **WARNING:** Linha ignorada (JSON malformado, campeonato inválido, etc.).
- **ERROR:** Falha na abertura de arquivos, problemas de I/O (estes sim abortam).

### 3.4 Relatório Final

Ao final do processamento, o programa logará um resumo:

```
Processamento concluído:
  - Linhas lidas: 1.000.000
  - Linhas ignoradas (JSON inválido): 42
  - Linhas ignoradas (campeonato fora do filtro): 150.000
  - Clubes escritos: 849.958
  - Jogadores escritos: 12.500.000
```

---

## 4. Escolha e Justificativa das Bibliotecas

### 4.1 Princípio: Somente Biblioteca Padrão

O enunciado permite qualquer biblioteca, mas a natureza do problema **não exige dependências externas**. Usar somente a stdlib do Python tem vantagens:

1. **Zero setup:** Não precisa de `pip install`, `venv`, ou `requirements.txt`.
2. **Portabilidade:** Roda em qualquer máquina com Python 3.8+.
3. **Confiabilidade:** Módulos da stdlib são mantidos pelo core team e não sofrem breaking changes.
4. **Demonstra domínio:** Mostra que o desenvolvedor conhece as ferramentas fundamentais.

### 4.2 Módulos Utilizados

| Módulo     | Uso                                      | Justificativa                                                                 |
|------------|------------------------------------------|-------------------------------------------------------------------------------|
| `json`     | Parse de cada linha JSONL                | `json.loads()` para parse individual. Não usa `json.load()` (carregaria tudo) |
| `csv`      | Escrita dos arquivos CSV                 | Implementa **RFC 4180** nativamente: escapa vírgulas, aspas, quebras de linha  |
| `sys`      | Receber caminho do arquivo via CLI       | `sys.argv[1]` — requisito explícito do enunciado                              |
| `os`       | Manipulação de caminhos de arquivo       | `os.path.join`, `os.path.dirname` para determinar diretório de saída          |
| `datetime` | Validação e formatação de datas          | `datetime.strptime()` para parse + validação em um só passo                   |
| `logging`  | Registro de erros e informações          | Sistema robusto de logging com níveis, sem `print()` espalhado pelo código    |

### 4.3 Por que NÃO usamos pandas?

| Aspecto      | pandas                                    | csv (stdlib)                              |
|--------------|-------------------------------------------|-------------------------------------------|
| Memória      | Carrega tudo em um DataFrame na RAM       | Streaming: O(1) por linha                 |
| Instalação   | Requer `pip install pandas` (~30MB)       | Zero dependências                         |
| Overhead     | Excessivo para transformação simples      | Leve e direto                             |
| RFC 4180     | `DataFrame.to_csv()` pode não escapar 100%| `csv.writer()` é RFC 4180 compliant       |
| Para este caso | **Overengineering**                      | **Ferramenta certa para o trabalho**       |

### 4.4 Compliance RFC 4180

O módulo `csv` do Python, quando usado com as configurações default, já segue o RFC 4180:

```python
writer = csv.writer(file)  # dialect='excel' por padrão
```

Comportamentos garantidos:
- Campos com vírgula → envoltos em aspas duplas: `"Pedro Lourenço, Filho"`
- Campos com aspas → aspas duplicadas: `"Ele disse ""olá"""`
- Campos com quebra de linha → envoltos em aspas duplas
- Separador: vírgula
- Terminador de linha: `\r\n` (CRLF, conforme RFC 4180)

> **Nota sobre o teste da amostra:** O campo `president` do Cruzeiro é `"Pedro Lourenço, Filho"` — contém vírgula! Este é um teste implícito da correta escrita CSV.

---

## 5. Mapeamento Completo: JSON → CSV

### 5.1 clubs.csv

```python
CLUBS_HEADER = [
    "Id do Clube",
    "Nome",
    "Campeonato",
    "Data de Fundação",
    "Cidade",
    "Estado",
    "País",
    "Estádio",
    "Presidente",
    "Apelido",
    "Cores"
]

def transform_club(record):
    return [
        safe_str(record, 'club_id'),
        safe_str(record, 'name'),
        safe_str(record, 'championship'),
        format_date(record.get('founding_date')),
        safe_str(record, 'city'),
        safe_str(record, 'state'),
        safe_str(record, 'country'),
        safe_str(record, 'stadium'),
        safe_str(record, 'president'),
        safe_str(record, 'nickname'),
        format_colors(record.get('colors')),
    ]
```

### 5.2 players.csv

```python
PLAYERS_HEADER = [
    "Id do Clube",
    "Id do Jogador",
    "Nome",
    "Idade",
    "Gols",
    "Data de Estreia",
    "Posição",
    "Número da Camisa"
]

def transform_player(club_id, player):
    return [
        club_id,
        safe_str(player, 'player_id'),
        safe_str(player, 'name'),
        safe_str(player, 'age'),
        safe_str(player, 'goals'),
        format_date(player.get('debut_date')),
        safe_str(player, 'position'),
        safe_str(player, 'shirt_number'),
    ]
```

---

## 6. Funções Utilitárias Críticas

### 6.1 `safe_str` — Extração Segura de Campos

```python
def safe_str(record, key):
    """Retorna o valor como string, ou '' se ausente/None."""
    value = record.get(key)
    if value is None:
        return ''
    return str(value)
```

### 6.2 `format_date` — Validação e Formatação de Datas

```python
from datetime import datetime

DATE_FORMATS = ['%Y-%m-%d', '%d/%m/%Y', '%Y/%m/%d']

def format_date(value):
    """Formata data para yyyy-MM-dd. Retorna '' se inválida."""
    if not value or not isinstance(value, str):
        return ''
    for fmt in DATE_FORMATS:
        try:
            return datetime.strptime(value.strip(), fmt).strftime('%Y-%m-%d')
        except ValueError:
            continue
    return ''
```

### 6.3 `format_colors` — União de Lista de Cores

```python
def format_colors(colors):
    """Une lista de cores com '|'. Retorna '' se vazia/ausente."""
    if not colors or not isinstance(colors, list):
        return ''
    return '|'.join(str(c) for c in colors if c is not None)
```

### 6.4 `is_valid_championship` — Filtro de Campeonato

```python
VALID_CHAMPIONSHIPS = {'serie a', 'serie b'}

def is_valid_championship(record):
    """Verifica se o campeonato é SERIE A ou SERIE B (case-insensitive)."""
    championship = record.get('championship')
    if not championship or not isinstance(championship, str):
        return False
    return championship.strip().upper().replace('É', 'E') in {'SERIE A', 'SERIE B'}
```

> **Nota:** A normalização `upper().replace('É','E')` trata variações como `"Série A"`. Porém, dado que o enunciado usa `"SERIE A"`, manter uma comparação case-insensitive simples pode ser suficiente. Esta é uma decisão defensiva.

---

## 7. Estrutura de Diretórios do Projeto

```
bigdatacorp/
├── spec/
│   ├── docs/
│   │   ├── enunciado_desafio.pdf
│   │   └── sample_clubes.jsonl
│   ├── research.md          ← Este documento
│   └── ia-usage.md          ← Log de uso de IA
├── src/
│   ├── main.py              ← Ponto de entrada (CLI)
│   ├── transformer.py       ← Regras de negócio e transformação
│   └── validator.py         ← Validação e parsing seguro
├── output/
│   ├── clubs.csv            ← Gerado pelo programa
│   └── players.csv          ← Gerado pelo programa
├── tests/
│   ├── test_transformer.py  ← Testes unitários das regras
│   └── test_validator.py    ← Testes unitários de validação
├── README.md                ← Instruções de uso
└── .gitignore
```

---

## 8. Cenários de Teste (Edge Cases)

| # | Cenário                                           | Comportamento Esperado                                |
|---|---------------------------------------------------|-------------------------------------------------------|
| 1 | Linha com JSON malformado (`{broken`)             | Logada como warning, ignorada                         |
| 2 | Linha vazia ou whitespace-only                    | Ignorada silenciosamente                              |
| 3 | `championship` diferente de SERIE A/B             | Clube e jogadores descartados                         |
| 4 | `nickname` é `null`                               | Campo vazio no CSV                                    |
| 5 | `nickname` está ausente (chave não existe)        | Campo vazio no CSV                                    |
| 6 | `colors` é lista vazia `[]`                       | Campo vazio no CSV                                    |
| 7 | `colors` é `null`                                 | Campo vazio no CSV                                    |
| 8 | `founding_date` com formato inválido `"abc"`      | Campo vazio no CSV; clube continua no arquivo          |
| 9 | `president` contém vírgula (`"Fulano, Jr."`)      | Envolto em aspas duplas pelo csv.writer                |
| 10| Clube da SERIE B sem jogadores                    | Aparece em `clubs.csv`, nenhuma linha em `players.csv` |
| 11| Campo numérico (age, goals) como `null`           | Campo vazio no CSV                                    |
| 12| `players` ausente ou `null`                       | Clube em `clubs.csv`, sem linhas em `players.csv`     |
| 13| Registro JSON válido mas não é dict (ex: `"abc"`) | Logado e ignorado                                     |

---

## 9. Considerações de Performance

| Aspecto                  | Abordagem                                                       |
|--------------------------|------------------------------------------------------------------|
| Leitura do arquivo       | Iterador nativo do Python (`for line in file`) — buffer interno |
| Parse JSON               | `json.loads()` por linha — não acumula                           |
| Escrita CSV              | `csv.writer.writerow()` — flush implícito por linha             |
| Memória                  | O(1) — proporcional ao maior registro individual, não ao total  |
| I/O                      | Sequencial. Para volumes extremos, poderia ser paralelizado, mas a stdlib é suficiente para milhões de registros |
| Estimativa               | ~500K–1M linhas/segundo com Python puro (depende do hardware)  |

---

## 10. Próximos Passos

1. **[Missão 2]:** Implementar o código-fonte seguindo esta arquitetura.
2. **[Missão 3]:** Gerar os CSVs a partir da amostra e validar contra os resultados esperados.
3. **[Missão 4]:** Escrever `README.md` com instruções de uso.
4. **[Missão 5]:** Preparar repositório Git para entrega.
