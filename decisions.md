# 📋 Registro de Decisões Arquiteturais (ADRs)

> **Projeto:** Desafio Técnico BigDataCorp  
> **Data:** 2026-08-09  
> **Formato:** Baseado em [Michael Nygard's ADR format](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions)

---

## ADR-001: Uso exclusivo de bibliotecas nativas do Python (`json`, `csv`, `datetime`, `logging`, `sys`, `os`)

### Status
**Aceito**

### Contexto
O enunciado permite "a linguagem e as bibliotecas que preferir". Precisamos decidir entre:
- **Opção A:** Bibliotecas nativas (`json`, `csv`, `datetime`, `logging`).
- **Opção B:** Dependências externas como `pandas`, `csvkit`, `orjson`, `ijson`.

O problema envolve: parse de JSONL, transformação de dados, escrita de CSV no padrão RFC 4180, e processamento de milhões de registros em streaming.

### Decisão
**Usaremos exclusivamente módulos da biblioteca padrão do Python.**

### Justificativa

#### `csv` — Compliance RFC 4180 nativa e eficiente
O módulo `csv` do Python é implementado parcialmente em **C** (no CPython), o que lhe confere performance superior a qualquer implementação manual em Python puro. Mais importante: ele já implementa o padrão RFC 4180 nativamente:

| Requisito RFC 4180                              | Comportamento do `csv.writer`                                |
|-------------------------------------------------|--------------------------------------------------------------|
| Campos com vírgula são envoltos em aspas         | ✅ Automático com `QUOTE_MINIMAL`                            |
| Aspas internas são duplicadas (`"` → `""`)       | ✅ `doublequote=True` (padrão)                               |
| Campos com quebra de linha são envoltos em aspas | ✅ Automático                                                |
| Separador padrão é vírgula                       | ✅ `delimiter=','` (padrão)                                  |
| Terminador de linha é CRLF                       | ✅ Com `newline=''` no `open()`, usa `\r\n`                  |

Usar `pandas.to_csv()` ou escrita manual com `f.write()` exigiria reimplementar ou confiar em comportamentos menos documentados.

#### `json` — Parser robusto com C-bindings
O módulo `json` do CPython usa internamente o módulo `_json` escrito em C para as operações de decode. `json.loads()` é suficientemente rápido para milhões de registros. Alternativas como `orjson` (Rust) ou `ujson` (C) são ~3-5x mais rápidas, mas:
- Adicionam dependências externas.
- Exigem compilação nativa (podem falhar em alguns ambientes).
- O gargalo do nosso pipeline é **I/O**, não parsing.

#### `datetime` — Validação de datas com strptime
`datetime.strptime()` faz parse **e** validação em um único passo. Se a data for inválida, lança `ValueError`. Isso é exatamente o que precisamos para a regra de negócio: "data inválida → campo vazio".

#### `logging` — Observabilidade sem `print()`
O módulo `logging` oferece níveis (INFO, WARNING, ERROR), formatação padronizada com timestamps, e é thread-safe. `print()` espalhado pelo código é anti-pattern para um programa de produção.

### Consequências
- ✅ **Zero dependências externas.** O projeto roda com `python main.py arquivo.jsonl` — sem `pip install`, sem `venv`, sem `requirements.txt`.
- ✅ **Portabilidade máxima.** Qualquer ambiente com Python 3.8+ executa sem preparação.
- ✅ **Manutenibilidade.** A stdlib não sofre breaking changes entre versões menores do Python.
- ⚠️ **Trade-off:** Performance de parsing JSON ligeiramente inferior a `orjson`. Aceitável porque o pipeline é I/O bound.

---

## ADR-002: Diferenciação entre "Erro de Estrutura" e "Erro de Validação"

### Status
**Aceito**

### Contexto
O enunciado exige dois comportamentos distintos para dados problemáticos:
1. *"Registros inválidos ficam de fora do resultado"* — a linha inteira é descartada.
2. *"Se o valor não for uma data válida, deixe o campo vazio — a linha continua"* — o campo fica vazio, mas o registro é preservado.

Precisamos de uma taxonomia clara de erros para que o código saiba **quando descartar a linha** vs. **quando apenas limpar o campo**.

### Decisão
Classificaremos erros em duas categorias com tratamentos distintos:

```
┌──────────────────────────────────────────────────────────────────┐
│                    ERRO DE ESTRUTURA                             │
│                                                                  │
│  Definição: O registro inteiro é irrecuperável.                  │
│  Exemplos:                                                       │
│    • Linha não é JSON válido (json.JSONDecodeError)              │
│    • JSON parseado não é um dict (ex: é uma lista, string, int) │
│                                                                  │
│  Ação: DESCARTAR a linha inteira.                                │
│  Log: WARNING com número da linha e motivo.                      │
│  Onde: reader.py (antes de qualquer regra de negócio)            │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                    ERRO DE VALIDAÇÃO                             │
│                                                                  │
│  Definição: Um campo específico tem valor problemático,          │
│  mas o registro como um todo é aproveitável.                     │
│  Exemplos:                                                       │
│    • `founding_date` é "abc" — não parseia como data            │
│    • `nickname` é null ou ausente                                │
│    • `colors` não é uma lista                                    │
│    • `age` é null                                                │
│                                                                  │
│  Ação: CAMPO VAZIO (""), registro permanece no CSV.              │
│  Log: Silencioso (isso é comportamento esperado, não erro).      │
│  Onde: transformer.py (funções format_date, safe_str, etc.)      │
└──────────────────────────────────────────────────────────────────┘
```

### Justificativa
Esta separação é derivada diretamente do enunciado:
- **Robustez** (*"O programa não deve abortar por causa de um registro problemático"*) → Erros de Estrutura são isolados no Reader.
- **Datas** (*"Se o valor não for uma data válida, deixe o campo vazio — a linha continua"*) → Erros de Validação são absorvidos no Transformer, gerando campo vazio.
- **Campos** (*"Campos ausentes ou nulos viram campo vazio"*) → Erros de Validação tratados com `safe_str()`.

### Consequências
- ✅ O código tem regras claras: Reader trata Estrutura, Transformer trata Validação.
- ✅ Não há ambiguidade sobre quando descartar vs. quando limpar.
- ✅ O Transformer pode ser testado com dicts Python simples, sem precisar de JSONL.
- ✅ Os logs ficam significativos: WARNINGs apenas para registros descartados, não para campos nulos (que são comportamento normal).

---

## ADR-003: Uso de Generators (`yield`) para streaming de milhões de registros

### Status
**Aceito**

### Contexto
O arquivo real pode conter "muitos milhões de registros". A abordagem ingênua seria:

```python
# ❌ Anti-padrão: O(N) de memória
with open(path) as f:
    all_data = [json.loads(line) for line in f]  # Materializa tudo
for record in all_data:
    process(record)
```

Para 10 milhões de registros com ~500 bytes cada, isso consumiria **~5GB de RAM** — inaceitável.

### Decisão
**Usaremos generators (`yield`) em toda a cadeia de leitura e filtragem.**

```python
def read_jsonl(filepath):
    """Generator: lê uma linha por vez, faz yield de dicts válidos."""
    with open(filepath, 'r', encoding='utf-8') as f:
        for line_number, raw_line in enumerate(f, start=1):
            stripped = raw_line.strip()
            if not stripped:
                continue
            try:
                record = json.loads(stripped)
            except json.JSONDecodeError:
                continue
            if isinstance(record, dict):
                yield line_number, record
```

### Justificativa Técnica

#### Como funciona um generator internamente
Um generator em Python **suspende** sua execução ao atingir `yield`, preservando o estado local (variáveis, posição no loop) no frame do generator. Quando o consumidor chama `next()`, a execução **retoma** de onde parou. Isso significa:

1. **Apenas 1 registro existe em memória por vez.** O dict do registro anterior é coletado pelo garbage collector quando o próximo `yield` acontece (assumindo que não há referências externas).
2. **O arquivo é lido incrementalmente.** O `for line in f` do Python usa o iterador nativo do file object, que mantém um buffer de leitura de ~8KB e não carrega o arquivo inteiro.
3. **A composição é zero-copy.** Generators podem ser encadeados sem materialização intermediária:

```python
raw     = read_jsonl(path)               # Generator 1
filtered = filter_championship(raw)       # Generator 2 (consome G1 sob demanda)
for record in filtered:                   # Consumidor final
    write(record)
```

#### Comparação de memória

| Abordagem              | Memória para 10M registros (~500B cada) |
|------------------------|----------------------------------------:|
| Lista completa          |                              ~5.0 GB    |
| pandas DataFrame        |                              ~6.5 GB*   |
| **Generator (yield)**  |                          **~20 KB**     |

> *pandas adiciona overhead de ~30% sobre os dados brutos por causa de index, dtypes, e metadata interna.

### Consequências
- ✅ **Memória O(1)** — independente do tamanho do arquivo.
- ✅ **Código idiomático** — generators são o mecanismo natural de streaming em Python.
- ✅ **Composável** — novos filtros podem ser adicionados como generators intermediários.
- ✅ **Testável** — um generator pode ser testado com `list(read_jsonl(test_file))` em testes unitários.
- ⚠️ **Trade-off:** Generators são single-pass. Não podemos "voltar" e reler um registro. Para este caso, isso não é necessário.

---

## ADR-004: Diretório de saída dos CSVs no mesmo local do arquivo de entrada

### Status
**Aceito**

### Contexto
O enunciado exige que o "caminho do arquivo de entrada é um parâmetro do programa". Precisamos decidir onde os CSVs de saída serão escritos.

Opções:
- **A:** No mesmo diretório do arquivo de entrada.
- **B:** No diretório de trabalho atual (`cwd`).
- **C:** Em um diretório parametrizável via argumento CLI opcional.

### Decisão
**Opção C com fallback para A:** O programa aceitará um argumento opcional para o diretório de saída. Se não fornecido, os CSVs serão escritos no mesmo diretório do arquivo de entrada.

```
Uso: python main.py <arquivo_entrada> [diretório_saída]
```

### Justificativa
- Manter os CSVs junto ao JSONL é a expectativa mais natural para avaliadores.
- Um parâmetro opcional para o diretório de saída dá flexibilidade sem complexidade.
- Usar `cwd` (opção B) seria confuso: se o avaliador roda de `/home`, os CSVs apareceriam lá e não junto aos dados.

### Consequências
- ✅ Comportamento previsível — CSVs ficam junto ao arquivo fonte.
- ✅ Flexível — diretório de saída pode ser customizado.
- ✅ Simples — apenas `sys.argv[1]` obrigatório, `sys.argv[2]` opcional.

---

## ADR-005: Normalização case-insensitive do campo `championship` para filtragem

### Status
**Aceito**

### Contexto
A regra de negócio diz: *"gere dados apenas para clubes que disputam a Série A ou a Série B"*. Os valores na amostra são `"SERIE A"`, `"SERIE B"`, `"SEM CAMPEONATO"`. Mas na base real, podem existir variações:
- `"Serie A"`, `"serie a"`, `"SERIE A "` (com espaço trailing)
- `"Série A"` (com acento)

### Decisão
A comparação será **case-insensitive** com `strip()`, comparando contra os valores normalizados `{"SERIE A", "SERIE B"}`.

```python
def is_valid_championship(record):
    champ = record.get('championship')
    if not champ or not isinstance(champ, str):
        return False
    normalized = champ.strip().upper()
    return normalized in {'SERIE A', 'SERIE B'}
```

### Justificativa
- A amostra usa `"SERIE A"` (sem acento). O enunciado também usa "Série A" e "Série B" (com acento) no texto descritivo. Portanto, consideramos ambos.
- A normalização `strip().upper()` é defensiva e trata variações triviais sem custo computacional.
- **Não** normalizamos acentos (ex: `Série` → `Serie`) para evitar complexidade desnecessária. Se a base real usar acento, incluiremos `"SÉRIE A"` e `"SÉRIE B"` no set de validação.

### Consequências
- ✅ Tolera variações comuns de capitalização e espaçamento.
- ✅ Simples e sem dependências externas (sem `unicodedata.normalize()`).
- ⚠️ Se a base real usar variações com acento, o set pode ser expandido facilmente.

---

## ADR-008: Otimização de I/O (Buffer Tuning) e Garbage Collector

### Status
**Aceito** (Fase 7 - Hyper-Otimização)

### Contexto
Em escala de Terabytes, dois gargalos ocultos degradam severamente o throughput do pipeline:

**Gargalo 1 — I/O Físico (sys_write excessivos):**
O CPython abre arquivos de texto com um buffer padrão de ~8 KB (`io.DEFAULT_BUFFER_SIZE`). Para cada ~8 KB gravados, o runtime faz uma chamada de sistema `write()` ao kernel. Em um pipeline que processa milhões de registros, isso gera milhões de syscalls desnecessárias. Em discos de rede (NFS, EFS, CIFS), cada syscall paga latência de rede, tornando o problema exponencialmente pior.

**Gargalo 2 — Pressão no Garbage Collector (GC):**
O loop principal cria e descarta milhões de dicts temporários (um por registro JSONL). O GC geracional do CPython monitora objetos na geração 0 e dispara coletas automáticas a cada ~700 alocações. Para 1 milhão de registros, isso equivale a ~1.400 coletas automáticas — cada uma causando uma micro-pausa (stop-the-world) que degrada o throughput de CPU.

### Decisão
**Buffer Tuning:** Aumentar o buffer de I/O para 256 KB (`buffering=262144`) em todas as chamadas `open()` do Reader e do Writer. Isso reduz as syscalls em ~32x (de 8 KB para 256 KB por chamada).

**GC Disable:** Desativar o Garbage Collector (`gc.disable()`) durante o loop principal de processamento e reativá-lo no bloco `finally`. Isso é seguro porque:
- O pipeline é single-threaded (sem referências cíclicas de outras threads).
- Os dicts temporários são efêmeros e serão coletados pelo reference counting normal do CPython (sem necessidade do GC geracional).
- O `finally` garante que o GC será reativado mesmo em caso de exceção.

### Consequências
- ✅ Redução de ~32x nas syscalls de I/O — impacto medível em discos de rede (EFS/NFS).
- ✅ Eliminação de ~1.400 coletas de GC por milhão de registros — throughput de CPU mais estável.
- ✅ Zero risco: o reference counting do CPython continua ativo; o `finally` garante reativação.
- ⚠️ O buffer de 256 KB aumenta o consumo de RAM em ~768 KB (3 arquivos × 256 KB) — desprezível.

---

## ADR-009: Padrão Dead Letter Queue (DLQ) para Auditoria de Dados

### Status
**Aceito** (Fase 7 - Auditoria)

### Contexto
Atualmente, quando o Reader encontra uma linha com JSON malformado ou tipo inesperado, ele loga um WARNING e descarta a linha. Em ambientes corporativos e financeiros, **"sumir" com o dado é inadmissível**. A equipe de Data Quality precisa inspecionar as linhas originais para:
- Diagnosticar problemas no sistema upstream (quem gerou o JSONL corrompido?).
- Quantificar a taxa de corrupção por arquivo.
- Reprocessar as linhas após correção.

### Decisão
**Implementar um arquivo de Dead Letter Queue (`dlq_errors.txt`) na pasta de output.**

O Reader receberá um file handle opcional de DLQ. Para cada linha rejeitada (JSONDecodeError ou tipo não-dict), a **string original bruta** (raw line) será gravada no arquivo DLQ, precedida por um prefixo com o número da linha e o motivo do descarte. O formato será:

```
[LINHA:4][JSON_MALFORMADO] isto nao e json
[LINHA:7][TIPO_INVALIDO:list] ["lista em vez de dict"]
```

### Consequências
- ✅ **Auditoria Completa:** Nenhum dado é perdido silenciosamente — toda linha rejeitada é preservada.
- ✅ **Rastreabilidade:** O prefixo `[LINHA:N]` permite localização imediata no arquivo original.
- ✅ **Reprocessamento:** A equipe de Data Quality pode extrair as linhas brutas e corrigir upstream.
- ✅ **Zero Impacto no Pipeline:** O pipeline continua processando normalmente; o DLQ é escrita paralela.
- ⚠️ Em datasets com alta taxa de corrupção, o arquivo DLQ pode crescer significativamente.
