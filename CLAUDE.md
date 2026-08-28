# Contexto do projeto — Pipeline de Learning Analytics (BCC701 / opCoders)

Este documento contextualiza um pipeline de dados educacionais que venho desenvolvendo em Python/Jupyter. Leia tudo antes de sugerir mudanças — há decisões de design que já foram validadas contra os dados reais.

## Visão geral

O projeto é ligado à Iniciação Científica do professor Reinaldo Silva Fortes (DECOM/UFOP), que avalia a eficácia de uma metodologia de ensino-aprendizagem para laboratórios remotos aplicada ao corretor automático **opCoders Judge**, usado na disciplina **BCC701 (Programação de Computadores I)**.

O pipeline que estou construindo funciona, na prática, como um **Learning Record Warehouse (LRW) sob medida**: unifica, para um período letivo por vez, três fontes de dados heterogêneas — submissões de código, logs de navegação no Moodle e registros acadêmicos oficiais — realizando limpeza, padronização, cruzamento e engenharia de atributos, para depois gerar análises de engajamento, procrastinação e rendimento.

Não há infraestrutura formal de **xAPI/LRS** por trás dos dados — cada fonte usa seu próprio formato e convenção, e o pipeline resolve essa unificação manualmente.

## Arquivos de dados de entrada

Todos são organizados **por período letivo** (ex.: `24-2`). Estrutura de pastas local esperada:

```
dados/
└── <período>/           # ex.: 24-2
    ├── full_data_merged.csv
    ├── dados_academicos.csv
    ├── <período>_mca_<professor>.csv
    └── Recursos Moodle e opCoders - <período> - <professor>.tsv
```

### 1. `full_data_merged.csv`
Histórico de submissões de código no opCoders Judge. Colunas principais: `delivery_id`, `user_id`, `class_name`, `task_question_id`, `task_name`, `question_name`, `grade_accounting`, `correction_error`, `grade`, `delivery_timestamp`, `task_starting_timestamp`, `task_finishing_timestamp`, `cod_curso`, `media_final`, `exame_especial`, `faltas`, `obs`, `sexo`, `data_nascimento`.

**Importante**: este arquivo já vem com os dados acadêmicos (`media_final`, `faltas`, `obs` etc.) mesclados linha a linha desde a origem — esse cruzamento foi feito **antes** da anonimização e **não é refeito pelo pipeline**.

### 2. `dados_academicos.csv`
Registro oficial da secretaria. Colunas: `id_aluno`, `sexo`, `data_nascimento`, `turma`, `ac1`, `pt1`, `ac2`, `pt2`, `cod_curso`, `media_final`, `exame_especial`, `faltas`, `obs` (APROV/REPRV), `professor`. Notas em formato brasileiro (vírgula decimal). **Não há coluna de período** — a distinção de semestre é feita pela pasta/arquivo, não por uma coluna interna.

### 3. `<período>_mca_<professor>.csv`
Matriz de navegação do Moodle. `id_anonimo` + ~56-116 colunas de recursos (uma por item: aula, material, exercício), preenchidas com status `"Concluído"`/`"Não concluído"` e timestamp de conclusão. **É gerado por professor** — um período pode ter vários professores, cada um com seu próprio arquivo MCA.

### 4. `Recursos_Moodle_e_opCoders_-_<período>_-_<professor>.tsv`
Dicionário de metadados que classifica cada recurso do Moodle em duas dimensões: `Conteúdo` (Informativo/Didático/AP/Avaliativo) e `Tópico` (Geral/PT1/PT2). Casa com o MCA **por nome exato de recurso** (coluna `Nome do recurso`). Também gerado por professor — cada professor pode ter recursos diferentes no seu Moodle.

### 5. `user_mapping.csv`
Relaciona `id_anonimo`/matrícula a outros identificadores do sistema. Não é lido diretamente pelas etapas centrais do pipeline (a anonimização já deve ter sido aplicada antes); serve de apoio para auditoria.

## Fato crítico sobre os identificadores (já verificado nos dados)

- `dados_academicos.id_aluno` e `<mca>.id_anonimo` são **o mesmo hash**, gerado a partir de `user_mapping.csv` — batem 100% entre si.
- `full_data_merged.user_id` usa **um esquema de anonimização diferente** — não bate com `id_aluno`, `id_anonimo` nem com `user_mapping.csv`. Isso não é um bug: o `full_data_merged.csv` já vem pré-cruzado com os dados acadêmicos (ver item 1 acima), então essa diferença de hash não afeta a análise.

## Regras de negócio do pipeline

1. **Padronização de turmas** (`numClassesFixed`): normaliza variações textuais de `class_name` via regex (ex.: `"Turma 11 - BCC104 (24.2)"` → `"Turma 11 (24.2)"`).
2. **Auditoria de IDs**: inner join entre `dados_academicos.id_aluno` e `<mca>.id_anonimo`, com relatório de perda (quantos alunos ficaram de fora de cada lado). No semestre 24-2: 365 alunos na secretaria, 85 no MCA do Reinaldo, 76 cruzados com sucesso.
3. **Saneamento decimal**: strings com vírgula → float; NaN nas notas parciais (`ac1`, `ac2`, `pt1`, `pt2`) → `0.0`.
4. **Mescla do exame especial**: `media_final = max(media_final, exame_especial)`.
5. **Notas bimestrais**: `P1 = 0.2*ac1 + 0.8*pt1`, `P2 = 0.2*ac2 + 0.8*pt2`.
6. **SSR (Submission Success Rate)**: indicador contínuo [0,1] de prontidão temporal da submissão — 0 = entrega na abertura da tarefa, 1 = entrega no prazo limite.
7. **Agregação de cliques do Moodle**: soma de recursos concluídos por aluno, com quebra por `Conteúdo` e `Tópico` usando o dicionário TSV (casamento por nome exato de coluna/recurso). No 24-2/Reinaldo: 54 de 56 recursos casaram (`Bibliografia básica` e `Material Didático Unificado` ficam de fora, contam só no total).
8. **Matriz de submissões pivotada**: tarefas × turmas, volume de submissões válidas (`grade_accounting`).
9. **Boxplots de notas filtrados por faltas** (limites regimentais, ex.: ≤35, ≤100).
10. **Scatter plots desempenho × volume de entregas**: 4-5 perfis dinâmicos (aprovado empenhado / aprovado baixo empenho / aprovado com falta / reprovado por nota / reprovado por falta), com linha de corte na média 6 e linha na média de entregas do grupo.
11. **Exportação multiformato**: `_intl.csv` (padrão internacional) e `_sheets.csv` (Google Sheets BR: `;` + vírgula decimal).

## Suporte a múltiplos professores por período

Um período pode ter mais de um professor lecionando a disciplina — e cada professor gera seu próprio par MCA + TSV. A estrutura atual usa:

```python
PROFESSORES_MCA = {
    "NOME_PROFESSOR": {"mca": caminho_do_mca, "tsv": caminho_do_tsv},
    # uma entrada por professor
}
```

Uma célula dedicada de **consolidação** lê todos os professores listados, concatena os MCAs (adicionando coluna `professor` para rastreabilidade, com aviso se algum `id_anonimo` aparecer em mais de um MCA — possível repetência) e os TSVs (com `drop_duplicates`), gravando os resultados como `PATH_LOGS_MOODLE` e `PATH_MAP_RECURSOS`. Isso mantém as células de análise seguintes **inalteradas** — elas continuam lendo essas duas variáveis como se houvesse um único professor.

Para o semestre 24-2, há apenas um professor (Reinaldo) — o dicionário tem uma única entrada.

## Problemas técnicos já identificados e resolvidos

- **`full_data_merged_numClassesFixed.xlsx`** (se aparecer): está corrompido — o Excel converteu as colunas `grade`, `media_final`, `exame_especial` em datas ao salvar. **Nunca usar esse `.xlsx` como fonte** — sempre regenerar a padronização de turmas a partir do `full_data_merged.csv` bruto.
- **Kaleido**: a versão `0.2.1` (usada originalmente, pensada para Colab) é incompatível com as versões atuais do Plotly, que exigem Kaleido `>=1.0.0`. Kaleido v1+ **requer o Google Chrome instalado localmente** para exportar imagens (`fig.write_image(...)`) — rodar `plotly_get_chrome` uma vez após instalar as dependências.
- **Ambiente Colab → local**: o notebook original tinha `from google.colab import drive` / `drive.mount(...)` e caminhos absolutos `/content/drive/MyDrive/...`. Na versão local, isso foi substituído por `BASE_DIR = Path("./dados")` e uma estrutura de pastas relativa por período.
- **Célula de instalação `pip install -U kaleido==0.2.1`** (sem `!`): funciona no Colab por reconhecimento automático de comandos, mas é erro de sintaxe em Python puro — precisa ser reescrita como `subprocess.run([sys.executable, "-m", "pip", "install", ...])`.

## Estado atual do código

- Notebook de produção: baseado em `Pipeline_Analise_Educacional_v2.ipynb`, evoluído para v3 com a lógica de múltiplos professores.
- Versão local (sem Colab, testada ponta a ponta com os dados reais do 24-2): `Pipeline_Analise_Educacional_local.py` (formato Jupytext `percent`, células marcadas com `# %%`, compatível com a extensão Jupyter/Python do VSCode) e seu par `.ipynb` equivalente.
- O teste local rodou com sucesso até a etapa de exportação de imagem (parou só por falta do Chrome, que é setup local, não bug de código). A auditoria de IDs bateu exatamente com os números esperados (365/85/76).

## O que eu quero que você faça

Estou migrando este projeto para trabalhar com Claude Code no VSCode. Preciso que você:
1. Trabalhe a partir do arquivo `Pipeline_Analise_Educacional_local.py` (ou do `.ipynb` equivalente) como base atual do pipeline.
2. Preserve todas as regras de negócio e decisões de design listadas acima ao propor mudanças — elas já foram validadas contra os dados reais.
3. Ao adicionar suporte a novos períodos (25-1, 25-2, 26-1), siga o padrão de `PROFESSORES_MCA` já estabelecido.
4. Sempre que mexer em algo que toque nos identificadores (`id_aluno`, `id_anonimo`, `user_id`), tenha em mente a distinção explicada acima — são dois esquemas de anonimização diferentes, e isso é esperado, não um erro a "corrigir".
