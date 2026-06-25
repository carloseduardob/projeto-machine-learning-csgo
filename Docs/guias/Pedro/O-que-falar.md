# Roteiro de estudo — sua parte da apresentação

Você cobre **Parte I + Parte II** (slides 1–12) e no Colab mostra **Partes 00, 01 e 02** (sem modelagem). Tempo sugerido: **~8–10 min** no total.

---

## Mapa mental (guarde isso)

```
CS:GO → problema real → dataset Kaggle
         ↓
       EDA (o que os dados mostram)
         ↓
    referências (quem já fez)
         ↓
    filtros anti-leakage → features → pipeline sklearn
         ↓
    demo Colab (00 → 01 → 02)
```

**Frase de ouro:** *“Não prevemos o futuro do jogo — usamos o estado do round no início para estimar quem tem mais chance de vencer.”*

---

# FASE 1 — Estudar o jogo (30 min)

## O que é CS:GO em 1 minuto

- Jogo **5 vs 5**: **CT** (defesa) vs **T** (ataque).
- Partida = vários **rounds** (~115 segundos cada).
- Ganha quem mata o time adversário, planta/desarma bomba ou estoura o tempo.
- **Economia virtual**: ganha $ por kill, vitória, objetivo → compra armas no round seguinte.

## Por que isso importa pro ML?

| No jogo | No nosso projeto |
|---------|------------------|
| Round | 1 linha do dataset (snapshot) |
| Quem ganhou | `round_winner` → CT ou T |
| Dinheiro / armas | colunas `ct_money`, `t_weapon_*`, etc. |
| “Time mais rico ganha” | heurística que acerta só **60,5%** |

## Slide 4 — o que falar

> “Nossa unidade de análise é o **round**. No início dele, cada time tem economia e armamento. Hoje isso é lido na intuição do jogador. Nós queremos **quantificar** essa vantagem com dados.”

---

# FASE 2 — O problema de ML (20 min)

## Slide 5 — duas tarefas

| Tarefa | Tipo | Alvo | Pergunta |
|--------|------|------|----------|
| **Principal** | Classificação binária | `target_cls` | CT ou T vence? |
| **Complementar** | Regressão | `money_diff` | Quanto CT tem a mais que T em $? |

**Classificação binária** = só duas classes (0 ou 1).  
`target_cls = 1` se T vence, `0` se CT vence.

**Regressão** = prever número contínuo (`money_diff = ct_money − t_money`).

Você **não precisa** explicar modelos **neste bloco** (slides 1–12 / Colab 00–02) — só cite que a **Parte III** traz LR, RF, SVM e KNN.

## Slide 6 — hipótese e dataset

Números para decorar:

| Dado | Valor |
|------|-------|
| Linhas | **122.410** |
| Colunas | **97** |
| Nulos | **0** |
| Heurística “mais $” | **60,5%** |
| Balanceamento | **51% T / 49% CT** |
| Origem | Skybox AI Challenge (~700 demos, 2019–2020) |
| Fonte | Kaggle — Lillelund |

> “A regra ‘quem tem mais dinheiro ganha’ falha em quase 40% dos casos. Por isso usamos ML para juntar economia, mapa, armas e estado do jogo.”

---

# FASE 3 — EDA (45 min)

**EDA** = Exploratory Data Analysis — “abrir o dataset e entender antes de modelar”.

## O que a EDA responde

1. Os dados estão **limpos**? (nulos, outliers)
2. As classes estão **balanceadas**?
3. O que **influencia** a vitória?
4. Tem **armadilha** (leakage)?

## Slide 7 — achados principais

| Achado | O que significa | O que falar |
|--------|-------------------|-------------|
| 51% / 49% | Classes equilibradas | “Não precisamos de técnica especial por desbalanceamento extremo” |
| Economia | `money_diff` ajuda, mas não basta | “Corrobora os 60,5%” |
| Mapa, AWP, bomba | fatores táticos extras | “ML pode usar tudo isso junto” |
| Outliers ~8% | `money_diff` com cauda longa | “Mantivemos — rounds muito desbalanceados são informação real” |
| **Leakage** | vários snapshots por round | “Snapshot do fim do round ‘entrega’ o resultado” |

### Leakage — conceito mais importante da EDA

**Data leakage** = usar informação que só existiria **depois** do que queremos prever.

Exemplo: no fim do round, `time_left` é baixo, jogadores mortos, bomba plantada → o modelo “vê o futuro”.

**Solução:** filtrar `time_left >= 150` (início do round).

> “É a decisão metodológica mais importante do projeto. Sem isso, o modelo parece ótimo mas não serve na prática.”

## Gráficos da EDA (se perguntarem)

| Arquivo | Mostra |
|---------|--------|
| `01_distribuicao_vencedor.png` | CT vs T equilibrados |
| `02_economia.png` | dinheiro CT/T e `money_diff` por vencedor |
| `04_buy_tier.png` | taxa de vitória por fase de compra |
| `06_awp_impacto.png` | AWP muda probabilidade |
| `09_correlacao.png` | variáveis que andam juntas |

---

# FASE 4 — Referências (30 min) — Slide 8

## Quem é quem (memorize 4 nomes)

```
Skybox → criou os dados (competição de IA)
Lillelund → publicou no Kaggle (nosso CSV)
Hiemstra → Bayes no mesmo dataset (método diferente)
Ghosh / CSGOPredictor → LR e RF (inspiração de baselines)
```

## Tabela rápida para falar

| Trabalho | O que fez | Relação com nós |
|----------|-----------|-----------------|
| **Skybox AI Challenge** | Gravou ~700 demos profissionais | **Origem** dos dados |
| **Lillelund (Kaggle)** | Publicou 122k snapshots | **Dataset que usamos** |
| **Hiemstra** | Probabilidade Bayesiana | Mesmo problema, outro método |
| **Ghosh / CSGOPredictor** | LR e RF no GitHub | Inspirou baselines; nós ampliamos com SVM, KNN e regressão |

## Posicionamento do nosso grupo (feche o slide assim)

> “Não inventamos o problema. Nos baseamos no ecossistema Skybox/Kaggle e entregamos um **pipeline reproduzível no Colab**, com EDA de leakage, feature engineering e comparação sistemática de modelos clássicos.”

---

# FASE 5 — Pré-processamento (45 min)

## Slide 10 — filtros

| Filtro | Regra | Por quê |
|--------|-------|---------|
| `time_left >= 150` | início do round | anti-leakage |
| 5 CT e 5 T vivos | 5v5 | snapshot de round completo |
| excluir `de_cache` | mapa raro | poucos dados |
| **Resultado** | 122.410 → **33.365** | ~27% das linhas |

> “Descartamos snapshots ‘sujos’ ou que vazam o resultado. Ficamos com o estado do round quando a decisão tática ainda faz sentido.”

## Slide 11 — feature engineering

**Feature engineering** = criar colunas novas a partir das existentes.

| Grupo | Features | Ideia |
|-------|----------|-------|
| Econômicas | `money_diff`, `money_ratio` | vantagem de $ |
| Estado | `health_diff`, `armor_diff`, `players_alive_diff` | CT − T |
| Armamento | `ct_rifle_count`, `ct_has_awp` | poder de fogo |
| Buy tier | `ct_buy_tier`, `t_buy_tier` | eco / force / half / full / mixed |

### Buy tier — explicar sem complicar

Dinheiro do time ÷ 5 jogadores → categoria de compra:

| Tier | Média $/jogador | Significado |
|------|-----------------|-------------|
| **force** | &lt; 1.700 | compra forçada barata |
| **eco** | &lt; 2.000 | economizando |
| **half** | 2.800–3.799 | meia compra |
| **full** | ≥ 4.700 | compra cheia |
| **mixed** | entre faixas | transição |

## Slide 12 — pipeline sklearn

Fluxo para decorar:

```
Dado bruto → filtros → features → ColumnTransformer → split 80/20
```

| Componente | O que faz | Por quê |
|--------------|-----------|---------|
| **StandardScaler** | z = (x − μ) / σ | KNN e SVM usam distância — precisam de escala igual |
| **OneHotEncoder** | mapa e buy tier viram colunas 0/1 | texto/categoria sem ordem falsa |
| **train_test_split 80/20** | 80% treino, 20% teste | avaliar em dados nunca vistos |
| **stratify=y** | mantém proporção CT/T | classes balanceadas no split |
| **GridSearchCV** | busca melhor hiperparâmetro | Pedro explica na Parte 3 |

---

# FASE 6 — Colab: o que mostrar e o que dizer

Abra: `COLAB_PROJETO-COMPLETO.ipynb` → role até a Parte 02 (não entre na 03).

## PARTE 00 — Setup (~1 min, só citar)

> “O notebook clona o repo, instala dependências e baixa o dataset do Kaggle automaticamente com `kagglehub` — sem precisar de conta Kaggle.”

**Sintaxe que pode aparecer:**

```python
kagglehub.dataset_download("christianlillelund/csgo-round-winner-classification")
pd.read_csv(CSV_PATH)
df.shape   # (122410, 97)
```

**O que dizer:** “122 mil linhas, 97 colunas, zero nulos.”

---

## PARTE 01 — EDA (~3 min)

### Célula de features na EDA

```python
df["money_diff"] = df["ct_money"] - df["t_money"]
df["target"] = (df["round_winner"] == "T").astype(int)
df["richer_wins"].mean()   # ~60.5%
```

| Código | Significado |
|--------|-------------|
| `df["col"] = ...` | cria coluna nova |
| `.astype(int)` | CT/T vira 0/1 |
| `.mean()` | média → proporção (ex.: 0.605 = 60,5%) |
| `.value_counts()` | conta CT vs T |

### Gráficos — uma frase cada

- **01** — “Classes equilibradas.”
- **02** — “Economia diferente entre vencedores.”
- **04** — “Buy tier muda chance de vitória.”
- **07** — “Bomba plantada favorece T.”
- **09** — “Correlação entre variáveis-chave.”

### Leakage no notebook

> “Aqui documentamos o risco: vários snapshots por round. Por isso na Parte 02 filtramos `time_left >= 150`.”

---

## PARTE 02 — Pré-processamento (~3 min)

### Filtros — código-chave

```python
mask = df["time_left"] >= 150
mask &= df["ct_players_alive"] == 5
mask &= df["t_players_alive"] == 5
mask &= ~df["map"].isin(MAP_EXCLUDE)
df.loc[mask]   # só linhas que passam
```

| Sintaxe | Significado |
|---------|-------------|
| `mask &= condição` | E lógico — todas as condições |
| `~df["map"].isin(...)` | exclui mapas da lista |
| `.loc[mask]` | filtra linhas |

**Mostre o print:**

```
Bruto: (122410, 97)
Após filtros: (33365, ...)
```

### Features derivadas

```python
"target_cls": (out["round_winner"] == "T").astype(int)
"ct_rifle_count": out[RIFLE_COLS_CT].sum(axis=1)
```

| Sintaxe | Significado |
|---------|-------------|
| `.sum(axis=1)` | soma por linha (total de rifles do time) |
| `.map(buy_tier)` | aplica função em cada valor |
| `pd.concat` | junta colunas novas ao DataFrame |

### Pipeline

```python
ColumnTransformer([
    ("num", StandardScaler(), numeric),
    ("cat", OneHotEncoder(...), categorical),
])
train_test_split(..., test_size=0.2, stratify=y_cls)
preprocessor.fit_transform(X_train)   # treino: aprende + transforma
preprocessor.transform(X_test)      # teste: só transforma
```

**Regra de ouro:** `fit` só no treino; `transform` no teste (evita vazar informação do teste).

**Pare aqui** antes da Parte 03:

> “Até aqui preparamos os dados. Na Parte 03 o Pedro treina os modelos com esse pipeline.”

---

# Roteiro falado — slide a slide (~8 min)

| Slide | Tempo | Fala resumida |
|-------|-------|---------------|
| **1** | 30s | Apresentação + título + Colab no GitHub |
| **2** | 20s | “Dividimos em 3 partes; eu cubro problema, EDA e pré-processamento” |
| **4** | 1min | CS:GO, round, economia, decisão subjetiva hoje |
| **5** | 45s | Classificação (CT/T) + regressão (`money_diff`) |
| **6** | 1min | 60,5%, dataset Kaggle, 122k×97, Skybox |
| **7** | 1min30 | EDA, balanceamento, fatores, leakage, outliers |
| **8** | 1min | Skybox, Lillelund, Hiemstra, Ghosh + nosso posicionamento |
| **9** | 10s | “Agora como preparamos os dados” |
| **10** | 1min | Filtros, 122k→33k |
| **11** | 1min | Features derivadas, buy tier |
| **12** | 45s | Scaler, OneHot, split 80/20 |
| **Colab** | 3–4min | Scroll 00→01→02, prints e 1–2 gráficos |

---

# Cola de sintaxe Python/pandas (não precisa decorar tudo)

```python
df.shape              # (linhas, colunas)
df.isna().sum()       # contagem de nulos
df["col"].mean()      # média
df.value_counts()     # contagem por categoria
df[col_a] - df[col_b] # diferença coluna a coluna
.astype(int)          # vira 0/1
.loc[mask]            # filtra linhas
.map(funcao)          # aplica função
.sum(axis=1)          # soma horizontal (por linha)
```

---

# Perguntas que o professor pode fazer

| Pergunta | Resposta curta |
|----------|----------------|
| Por que ML e não regra simples? | 60,5% não basta; dezenas de variáveis e relações não lineares |
| O que é leakage? | Snapshot do fim do round revela o resultado; filtramos início |
| Por que 122k virou 33k? | Filtros de qualidade e anti-leakage |
| Por que StandardScaler? | KNN/SVM usam distância; escalas diferentes distorcem |
| Por que OneHot no mapa? | Mapa é categoria, não número ordenado |
| De onde veio o dataset? | Skybox Challenge → Lillelund no Kaggle |
| O que vocês fizeram de novo? | Pipeline Colab reproduzível + comparação sistemática de modelos clássicos |

---

# Plano de estudo (3 dias)

### Dia 1 — Conceitos
- [ ] Ler slides 4–6
- [ ] Assistir 5 min de CS:GO no YouTube (round + economia)
- [ ] Decorar: 122.410, 33.365, 60,5%, 51/49

### Dia 2 — EDA e referências
- [ ] Slides 7–8
- [ ] Explicar leakage para alguém em 1 frase
- [ ] Ler `Docs/slides/referencias-e-trabalhos-relacionados.md`

### Dia 3 — Pré-processamento + Colab
- [ ] Slides 10–12
- [ ] Abrir Colab, rodar até Parte 02
- [ ] Ensaiar: slides → Colab → “passo para o Pedro na Parte 03”

---

# Transição para o Pedro (memorize)

> “Preparamos o dataset com filtros anti-leakage, criamos features táticas e montamos o pipeline no scikit-learn. No Colab, isso corresponde às Partes 00, 01 e 02. Na **Parte III** entram os modelos treinados e os resultados.”

Se quiser, no próximo passo monto um **script palavra por palavra** só da demo do Colab ou um **quiz de 10 perguntas** para você testar antes da apresentação.