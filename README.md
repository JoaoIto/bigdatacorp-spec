# Especificação e Planeamento: Desafio Técnico BigDataCorp (Batch)

Este repositório contém a fase de pesquisa profunda, desenho de arquitetura e planeamento tático para a resolução do Desafio Técnico de Analista de Back-end Pleno (Equipa de Processamento em Lote) da BigDataCorp. 

O objetivo destas especificações é estabelecer as fundações para um pipeline de dados I/O Bound capaz de processar milhões de registos, garantindo complexidade de espaço $O(1)$, conformidade estrita com o padrão RFC 4180 e elevada tolerância a falhas face a dados malformados.

## Sumário

1. [Estrutura da Documentação](#1-estrutura-da-documentação)
2. [Visão Geral do Sistema](#2-visão-geral-do-sistema)
3. [Estratégia de Resiliência](#3-estratégia-de-resiliência)
4. [Auditoria e Utilização de IA](#4-auditoria-e-utilização-de-ia)

---

## 1. Estrutura da Documentação

Os ficheiros presentes neste repositório refletem o processo de engenharia de software aplicado antes da escrita do código de produção. Cada documento tem um propósito específico no ciclo de vida do projeto:

* [**research.md**](./research.md): Análise exaustiva do problema. Contém o mapeamento estrutural de JSON para CSV, o inventário das regras de negócio (filtros, relacionamentos 1:N) e a identificação de casos extremos (edge cases) extraídos da amostra de dados.
* [**architecture.md**](./architecture.md): Desenho do pipeline de processamento. Detalha a implementação dos padrões *Pipes & Filters* e *Generator Pattern*, provando matematicamente a alocação de memória independente do volume total do ficheiro.
* [**decisions.md**](./decisions.md): Registo de Decisões Arquiteturais (ADRs). Documenta e justifica tecnicamente as escolhas de engenharia, nomeadamente a opção pelo uso exclusivo da biblioteca padrão (stdlib) de Python em detrimento de ferramentas baseadas em RAM (como Pandas).
* [**implementation-plan.md**](./implementation-plan.md): Roteiro tático de execução. Divide o desenvolvimento do código-fonte em fases progressivas (Infraestrutura, Core I/O, Regras de Negócio e Testes), com micro-tarefas mapeadas.
* [**ia-usage.md**](./ia-usage.md): Registo de auditoria. Documenta cronologicamente os *prompts* e as iterações arquiteturais concebidas com o auxílio de Inteligência Artificial, servindo como prova de utilização metódica e orientada a engenharia.
* [**docs/**](./docs/): Diretório reservado para o armazenamento dos artefactos originais do desafio (enunciado em formato PDF e ficheiro de amostra JSONL).

## 2. Visão Geral do Sistema

A solução foi projetada em três camadas lógicas distintas para garantir a separação de responsabilidades (SoC):

* **Reader (Leitura em Fluxo):** Consome o ficheiro JSONL através de *generators* (`yield`), garantindo que apenas um objeto é carregado para a memória em determinado instante.
* **Transformer & Validator (Processamento):** Aplica o filtro central de competições ("Série A" e "Série B"), formata datas estritamente para `yyyy-MM-dd` e normaliza vetores (arrays) utilizando pipes (`|`).
* **Writer (Escrita RFC 4180):** Encarrega-se da gravação assíncrona ou síncrona nos ficheiros relacionais `clubs.csv` e `players.csv`, lidando nativamente com o escape de caracteres especiais (vírgulas e aspas internas).

## 3. Estratégia de Resiliência

Para cumprir o requisito de robustez perante bases de dados reais, o sistema implementa uma taxonomia de erros estrita:
* **Erros de Validação:** Quando um dado não cumpre o formato esperado (exemplo: data de fundação inválida), o sistema anula o valor (campo vazio no CSV), sem descartar a entidade.
* **Erros de Estrutura:** Quando o registo apresenta falhas críticas (exemplo: JSON corrompido), a linha é descartada, o erro é registado no sistema de *logs* (nível ERROR) e a execução prossegue ininterruptamente para a linha seguinte. O processo nunca é abortado por anomalias em dados isolados.

## 4. Auditoria e Utilização de IA

Conforme as diretrizes da avaliação técnica, o uso de assistentes virtuais de código foi submetido a rigoroso controlo. A Inteligência Artificial foi empregue de forma arquitetural (discussão de *design patterns*, *edge cases* e estruturação de *ADRs*), sem a delegação cega de escrita de blocos monolíticos de código. O rasto completo destas interações encontra-se formalizado e acessível em `ia-usage.md`.
