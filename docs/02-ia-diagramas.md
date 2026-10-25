# 02 — Uso de IA Generativa neste Discovery

Este documento registra **como** a IA generativa foi usada para produzir os artefatos da
Unidade 3, o que foi **aproveitado**, **modificado** e **descartado** — e, principalmente,
**por quê**.

A postura adotada (a mesma da especificação de requisitos da Unidade 2): tratar o
assistente como um **analista júnior rápido e incansável, com acesso de leitura ao
código-fonte, mas sem acesso à cabeça do dono**. Nesta unidade, porém, o “material bruto”
não foi um documento de elicitação — foi o **próprio repositório**: o assistente recebeu o
conteúdo de `src/caption_dataset_for_lora.py`, `src/caption_gradio_app.py`, `scripts/`,
`workflows/`, `requirements.txt` e o README upstream, e foi pedido para produzir (a) a
descrição em linguagem natural, (b) um diagrama estrutural inspirado no C4 e (c) diagramas
de sequência das jornadas — todos em Mermaid.

---

## 1. Onde a IA foi usada

| Etapa | Papel da IA | Papel humano |
|---|---|---|
| Estrutura da descrição | Propôs a ordem de seções (escopo / nível / fronteiras / integrações / restrições / lacunas) | Definir o nível de visão correto (ferramenta de lote, não serviço) e corrigir vocabulário |
| Diagrama estrutural | Primeira redação da visão de containers | Verificar nó a nó e aresta a aresta contra o `src/`; cortar o que o modelo inventou (D-01..05) |
| Diagramas de sequência | Primeira redação das duas jornadas (Gradio e CLI) | Validar a ordem dos passos e a semântica real de erro (lido no código) |
| Registro de lacunas | Enumerar candidatas a lacunas | Confirmar cada uma contra o código e decidir quais são reais (LG-01..10) |
| “O que o agente precisaria ter” | Enumerar os contratos que um agente seria forçado a inventar | Decidir o que travar com valor e o que deixar como decisão aberta (doc 03) |

---

## 2. O que o modelo inferiu **corretamente**

### A-01 — Os três backends, e as duas naturezas de execução

O modelo distinguiu corretamente as duas classes: BLIP/Florence-2 são **in-process**
(`transformers`), enquanto Ollama é um **runtime externo** alcançado por HTTP — e amarrou
isso ao detalhe certo: é por isso que somente o Ollama aceita prompt custom e contexto de
arquivo/EXIF. Na primeira passada ele havia modelado os três como “bibliotecas de ML”
equivalentes (M-01).

### A-02 — Duas fachadas, um motor

Capturou que a UI Gradio é **apresentação** sobre `caption_folder()` — a mesma função que a
CLI chama — e não criou dois fluxos distintos no diagrama. Esse é o erro mais comum deste
tipo de diagrama (a UI “faz” a legendagem).

### A-03 — A semântica de progresso/ETA

Reproduziu exatamente a política real: ETA pela média dos **3 primeiros arquivos com
sucesso** (o docstring diz “first 3 successful times”), “provisional” com menos de 3, e
horário previsto de término. Não inventou barra percentual suave nem métricas inexistentes.

### A-04 — A fronteira de privacidade do EXIF

Manteve a whitelist, a exclusão de GPS e a instrução “confie nos pixels” como **propriedades
do sistema** — e colocou o opt-out (`--ollama-no-file-context` / checkbox) no nível certo:
decisão do usuário, não detalhe técnico.

### A-05 — A escolha da jornada crítica

Dado o leque (legendar, treinar, preparar dataset, baixar modelo), escolheu a **execução de
legendagem em lote** como jornada crítica — a única para a qual a ferramenta existe — e
ordenou os passos como no código: scan → contagem de trabalho (pulando `.txt` existentes)
→ carregamento lazy do modelo → loop por arquivo com **isolamento de erro por arquivo**.

### A-06 — A saída como contrato

Descreveu a fronteira *downstream* como “o dataset no formato que os treinadores consomem
(`folder_path` + `caption_ext: txt`)”, e não como “arquivos de imagem + texto” — o que torna
a relação com `ai-toolkit` / `ltx-video-trainer` significativa em vez decorativa.

---

## 3. O que precisou ser **modificado**

### M-01 — Ollama modelado como biblioteca

A primeira redação colocou Ollama **dentro** da fronteira do sistema como “biblioteca de
backend de ML”. **Modificado:** ele é um **sistema externo** (um processo que o usuário
precisa ter rodando), e o que fica dentro da fronteira é apenas o *adaptador* HTTP.
Consequência no diagrama: dois nós de backend distintos (HF in-process × Ollama HTTP) e um
nó externo de runtime Ollama.

### M-02 — ComfyUI modelado como integração

O rascunho mostrava o ComfyUI “recebendo o dataset” por uma seta de dados. **Modificado:**
não existe nenhuma chamada — o ComfyUI entra (a) como **ambiente Python** que os `.bat`
reutilizam, e (b) como morada dos workflows de exemplo. No diagrama, relação tracejada de
“fornece ambiente”, não fluxo de dados.

### M-03 — Nível do diagrama

O primeiro diagrama estrutural do modelo era de **componente** (funções internas de
`caption_folder`). O pedido era **containers**: subi a abstração para
CLI / Gradio / motor / backends / scripts / workflows, e rebaixei as funções internas para
texto *dentro* dos containers.

### M-04 — O rótulo de resize (o caso que o modelo **não** pegou)

O rótulo da UI é `Max side (px, 0=no resize; HF only)`, mas o loop aplica `resize(im,
max_side)` a **toda** imagem antes de chamar o backend — **Ollama incluído** (o que vai na
rede é a imagem já redimensionada, re-encodada em JPEG q90). O modelo, lendo a UI,
transportou “HF only” para o diagrama. **O código vence**: descrição e diagrama dizem
“borda maior redimensionada antes da legendagem, para todos os backends”. Este é um caso
real de divergência interface × código no sistema de origem, registrado como `LG-05`.

### M-05 — O fluxo de erro da sequência

O primeiro diagrama de sequência **encerrava a execução** na primeira falha. O código
**isola por arquivo**: linha `FAIL` com a exceção, contador avança, loop continua, e só o
tempo dos *sucessos* entra na ETA. **Modificado** com `alt/else` por arquivo.

### M-06 — O carregamento “lazy” do modelo

O rascunho carregava o modelo **antes** do loop. No código, `load_cap(...)` é chamado
**depois** do scan e da contagem de trabalho (e, para o Ollama, é só um *closure* — o custo
real aparece no primeiro request; nos modelos HF, o primeiro uso ainda é o download dos
pesos). **Modificado**: carregamento após validações, com nota explícita de “primeiro uso”.

---

## 4. O que foi **descartado**

### D-01 — Serviço web + banco + fila

A primeira redação veio com API REST, tabela `captions`, fila de jobs com workers e
Dockerfile. **Tudo descartado:** o sistema é uma ferramenta de lote local; não há
persistência além dos arquivos. Adicionar a arquitetura de um *serviço* seria **inventar
requisitos** — o mesmo padrão de falha da Unidade 2 (“o que sistemas assim costumam ter”).

### D-02 — Autenticação / multiusuário

Descartado junto do D-01; o único “usuário” é o operador da máquina, e o Gradio escuta em
`127.0.0.1` **próprio para não expor**.

### D-03 — Fallback em nuvem (OpenAI / Replicate)

Proposto como “se o Ollama local cair, chama uma API de nuvem”. **Descartado:** local-first
é princípio de design (privacidade, custo, sem chave). Documentar um fallback que não
existe seria **mentir na arquitetura**.
---

## 5. Balanço

| Categoria | Quantidade |
|---|---|
| Aproveitadas (inferências corretas) | 6 |
| Modificadas antes de aproveitar | 6 |
| Descartadas | 5 |

**O que a IA fez bem aqui:** dado o código como contexto, o modelo acertou a *forma* da
arquitetura (2 fachadas / 1 motor / 2 classes de backend / arquivos como único estado) e as
semânticas finas que eu provavelmente teria que caçar no código (ETA dos 3 primeiros,
skip/overwrite, whitelist de EXIF). O ganho, como na Unidade 2, foi de **cobertura e
velocidade da primeira redação**.

**Onde ela falhou do mesmo jeito da Unidade 2:**
1. **Preencher a fronteira com a arquitetura canônica de “ferramenta web”** (D-01/02/03) —
   a ferramenta é um pipeline de lote, e o modelo insistiu em transformá-la em serviço;
2. **Confiar na interface sobre o código** (M-04) — o rótulo da UI venceu o loop;
3. **Alisar o fluxo de erro** (M-05) — o rascunho tornava a execução *atômica* quando o
   código a torna *tolerante*.

**A lição viaja com a unidade:** diagrama gerado por modelo é tão bom quanto a
**verificação contra a fonte**. Os três `.mmd` deste repositório estão no estado em que
**todo nó e toda aresta rastreiam para uma linha de código** — e as divergências entre o
rascunho gerado e esse estado estão documentadas em M-01..06. É exatamente isso que torna
o diagrama uma entrada confiável para um agente de desenvolvimento no futuro.

---

## Anexo — Prompts que produziram os melhores resultados

1. *“Aqui está o código-fonte de X. Desenhe um diagrama de containers (estilo C4) em
   Mermaid. **Todo nó precisa ser algo que exista no código.** Onde tiver dúvida, coloque o
   item **fora da fronteira** e marque com uma interrogação.”* — a cláusula “marque com
   interrogação” é o que expôs D-01..04.
2. *“Das jornadas desta ferramenta, qual, se quebrar, torna a ferramenta inútil? Trace-a
   como diagrama de sequência usando **somente mensagens que o código efetivamente envia**
   (cite a função de cada).”*
3. *“Onde o rótulo da UI, o README e o código dão **três descrições diferentes** para o
   mesmo parâmetro?”* — foi o que trouxe a LG-05 (rótulo de `max_side`).
4. *“Liste todo comportamento que depende do **primeiro uso** (download, pull, cache) e o
   que o usuário precisa fazer antes de a ferramenta funcionar.”*


### D-04 — “Plugin do ComfyUI”

O rascunho descreveu a ferramenta como nó/plugin do ComfyUI. **Corrigido:** a pasta
`workflows/` contém **JSON de exemplo** (um com grafo de nós utilitários, outro só de
`Note`); a ferramenta **não registra nada** no ComfyUI nem o importa.

### D-05 — CI/CD, empacotamento, PyPI

O modelo ofereceu `pyproject.toml` + CI + publicação em PyPI como “próximos passos”.
**Descartado** da documentação *as-is*: o sistema **atual** é uma pasta de scripts;
empacotamento é **lacuna** (terreno de LG-08/LG-09), não componente.
