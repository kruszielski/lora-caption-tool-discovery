# lora-caption-tool — Discovery de Documentação · Unidade III (Diagrams as Code)

Este repositório concentra a **fase de discovery de documentação** de um sistema que o
aluno realmente usa: o
[`lora-caption-tool`](https://github.com/kruszielski/lora-caption-tool) — ferramenta de
lote **local** que gera **legendas densas (`.txt`)** para datasets de treino de **LoRA**,
usando modelos de visão (**BLIP**, **Florence-2** ou **Ollama**).

Na esteira da abordagem da unidade, **tudo aqui é código**: a descrição em linguagem
natural, os diagramas **Mermaid** (gerados com **GenAI** e depois **verificados linha a
linha contra o código-fonte**) e o registro das decisões sobre o que o modelo produziu.
O objetivo é que este repositório sirva, no futuro, de **contexto para agentes de
desenvolvimento** implementarem o sistema com aderência à arquitetura documentada.

> **Nada do sistema é reimplementado aqui.** O repositório concentra o discovery: a
> documentação e os diagramas são a entrega; o código continua no
> [repositório upstream](https://github.com/kruszielski/lora-caption-tool).

---

## O caso

`lora-caption-tool` resolve uma dor real do fluxo de treino de LoRA de personagem/estilo:
escrever uma legenda por imagem **à mão** é lento e inconsistente — e legenda
inconsistente vira ruído para o treinador. A ferramenta percorre uma pasta de imagens,
passa cada uma por um modelo de visão (local, com EXIF e nome de arquivo como contexto) e
grava `imagem.txt` ao lado — o formato exato que `ai-toolkit`, `ltx-video-trainer` e
ComfyUI esperam. Ela tem **duas fachadas** (CLI com `.bat` de drag & drop, e UI Gradio
local) sobre **um único motor** (`caption_folder`).

---

## Artefatos

| # | Artefato | Conteúdo |
|---|---|---|
| [01](docs/01-descricao-sistema.md) | **Descrição do sistema** (linguagem natural) | Escopo (dentro/fora), nível da visão, fronteiras e responsabilidades, integrações, restrições e **10 lacunas** registradas |
| [02](docs/02-ia-diagramas.md) | **Uso de GenAI** | O que o modelo inferiu corretamente, o que foi ajustado, o que foi descartado — com justificativa |
| [03](docs/03-contractos-para-agentes.md) | **Contratos para agentes** | As ~30 decisões que um agente inventaria se não estivessem travadas; 5 ficam `ABERTO` de propósito |
| [diagrams/](diagrams/) | **Fontes Mermaid** | `container-view.mmd` (estrutural), `sequence-gradio-ollama.mmd` e `sequence-cli-batch.mmd` (comportamentais) + PNGs renderizados |
| [AGENTS.md](AGENTS.md) | **Contexto para agentes de desenvolvimento** | O que o sistema é, o que **não** fazer, e a regra “número sem marcador é lacuna” |
| [POSTAGEM-FORUM.md](POSTAGEM-FORUM.md) | **Postagem do fórum** | texto pronto (versões completa e curta) |

---

## 1 · Os diagramas — renderizados

Ambos os tipos pedidos pela atividade estão **neste README, renderizados** (o GitHub
renderiza Mermaid nativamente) e versionados em [`diagrams/`](diagrams/): o bloco abaixo
**é** o arquivo `.mmd` — uma única fonte de verdade.

Eles foram gerados por um assistente de GenAI **com acesso ao código-fonte** da
ferramenta e depois verificados contra `src/` — o registro completo de correções está em
[docs/02](docs/02-ia-diagramas.md).

### 1.1 Estrutural — visão de containers (inspirada no C4)

```mermaid
%% Visão estrutural — visão de containers (inspirada no C4) do lora-caption-tool.
%% Gerada com GenAI a partir do código-fonte e validada linha a linha em src/
%% (registro de ajustes: docs/02-ia-diagramas.md, itens M-01..M-03).
%% Este arquivo É a fonte de verdade da arquitetura (diagrams-as-code).
flowchart TB
  classDef person fill:#0E7490,stroke:#155E75,color:#fff,stroke-width:2px;
  classDef container fill:#4488CC,stroke:#2F6599,color:#fff;
  classDef ext fill:#6B7280,stroke:#4B5563,color:#fff;
  classDef data fill:#C3AA68,stroke:#9C8440,color:#1F2937;

  user(["Operador<br/>(máquina única, Windows)"]):::person

  subgraph SYS["lora-caption-tool — ferramenta de lote local (sem serviço, sem banco, sem fila)"]
    bat["Lançadores Windows<br/>Caption_Dataset_For_LoRA.bat (arrastar pasta)<br/>atalhos .bat/.ps1"]:::container
    cli["Fachada CLI<br/>caption_dataset_for_lora.py<br/>(argparse)"]:::container
    ui["Fachada web (opcional)<br/>caption_gradio_app.py<br/>Gradio · 127.0.0.1:7866"]:::container
    eng["Motor — caption_folder()<br/>scan · resize · EXIF · progresso/ETA · grava .txt"]:::container
    hf["Backends HF (in-process)<br/>BLIP base/large · Florence-2<br/>(transformers)"]:::container
    ola["Adaptador Ollama (HTTP)<br/>urllib · POST /api/chat<br/>1 imagem por request · timeout 300s"]:::container
    wf["Workflows de exemplo<br/>workflows/*.json (ComfyUI)"]:::container
  end

  ollama["Ollama — runtime EXTERNO<br/>127.0.0.1:11434<br/>qwen3.6:27b (~17 GB)"]:::ext
  hub["Hugging Face Hub<br/>pesos BLIP ~1,5 GB / Florence-2 ~1,2 GB<br/>(download no 1º uso)"]:::ext
  comfy["ComfyUI Desktop<br/>fornece a .venv Python (não é chamada)"]:::ext
  trainers["Treinadores de LoRA<br/>ai-toolkit · ltx-video-trainer"]:::ext

  srcds[("Pasta do dataset<br/>imagens + EXIF")]:::data
  outds[("Dataset com legendas<br/>imagem + .txt sidecar")]:::data

  user -- "arrasta a pasta no .bat" --> bat
  bat -- "python src/caption_dataset_for_lora.py + args" --> cli
  user -- "navegador" --> ui
  ui -- "run_caption(...)" --> eng
  cli -- "caption_folder(...)" --> eng
  eng -- "imagem Pillow (redimensionada)" --> hf
  eng -- "JPEG q90 b64 + prompt (+ contexto EXIF)" --> ola
  ola -- "HTTP localhost:11434" --> ollama
  hf -. "1º uso: baixa pesos" .-> hub
  comfy -. "mesmo ambiente Python" .- bat
  eng -- "lê imagens / EXIF" --> srcds
  eng -- "grava .txt (UTF-8)" --> outds
  outds -. "pasta consumida como folder_path + caption_ext: txt" .-> trainers
  wf -. "pipeline de referência" .- trainers
```

**Como ler:** duas fachadas sobre **um** motor; duas *classes* de backend (in-process ×
HTTP externo); os **arquivos são o único estado** (não há banco); os sistemas externos
(Ollama, HF Hub, ComfyUI, treinadores) **não pertencem** ao sistema — e o ComfyUI,
notadamente, *fornece ambiente* em vez de receber dados
([M-01/M-02](docs/02-ia-diagramas.md)).

### 1.2 Comportamental — jornada crítica: legendagem em lote pela UI Gradio (Ollama)

```mermaid
%% Jornada crítica — legendagem em lote pela UI Gradio (backend Ollama).
%% Validada contra src/caption_gradio_app.py e src/caption_dataset_for_lora.py
%% (registro: docs/02-ia-diagramas.md, itens A-05, M-05, M-06).
sequenceDiagram
  autonumber
  actor U as Usuário (navegador)
  participant UI as UI Gradio 127.0.0.1:7866
  participant E as Motor caption_folder()
  participant O as Ollama 127.0.0.1:11434
  participant FS as Filesystem

  U->>UI: formulário: pasta, prefixo, backend=ollama, checkboxes → "Run captioning"
  UI->>E: caption_folder(input, output, prefix, max_side, on_progress, log)

  E->>FS: list_images(pasta, recursive)
  FS-->>E: imagens ordenadas (png/jpg/jpeg/webp/bmp/gif)
  E->>E: trabalho = imagens sem .txt (todas, se overwrite)

  alt pasta não existe
    E-->>UI: "ERROR: not a folder: …" → encerra
  else sem imagens / tudo já legendado
    E-->>UI: "No images …" / "No captions to write — use --overwrite" → encerra
  end

  Note over E,O: carregamento lazy — o custo real aparece no 1º request

  loop cada imagem (sequencial, 1 por vez)
    E->>FS: abre imagem (Pillow) → resize da borda máx. ≤ 1024 (LANCZOS)
    E->>E: contexto: basename + EXIF whitelist (sem GPS) — se o checkbox estiver ativo
    E->>O: POST /api/chat {prompt + contexto, JPEG q90 b64} · stream:false · 300s
    alt 200 OK com conteúdo
      O-->>E: texto da legenda
      E->>FS: escreve imagem.txt = prefix+legenda+suffix (UTF-8, "\n")
      E->>UI: on_progress: barra + ETA (média dos 3 primeiros OK)
    else URLError / HTTPError / resposta vazia
      O--xE: erro
      E->>UI: linha FAIL (exceção) — o loop CONTINUA
    end
  end

  E-->>UI: "Done. Wrote {ok} captions, skipped {sk}."
  UI-->>U: log completo na caixa + progresso 100%
```

**Como ler:** a jornada foi escolhida pela pergunta do [docs/02](docs/02-ia-diagramas.md)
— *qual, se quebrar, torna a ferramenta inútil?* É a única que ela existe para fazer.
Dois detalhes do código aparecem como **decisões de design** no diagrama: o **carregamento
lazy** do modelo (M-06) e o **erro isolado por arquivo** — a execução é *tolerante*, não
atômica (M-05).

### 1.3 Comportamental — jornada de drag & drop no Windows (CLI)

```mermaid
%% Jornada — drag & drop no Windows (fachada CLI) — arrastar a pasta no .bat.
%% Validada contra scripts/Caption_Dataset_For_LoRA.bat e main() em
%% src/caption_dataset_for_lora.py.
sequenceDiagram
  autonumber
  actor U as Usuário (Windows)
  participant BAT as Caption_Dataset_For_LoRA.bat
  participant CL as CLI main() na .venv do ComfyUI
  participant E as Motor caption_folder()
  participant B as Backend BLIP / Florence-2 / Ollama
  participant FS as Filesystem

  U->>BAT: solta a pasta do dataset (argumento 1)
  BAT->>CL: .venv\Scripts\python.exe src\caption_dataset_for_lora.py "<pasta>" [--prefix …]
  CL->>CL: normalize_path: remove aspas vindas do Explorer, expande ~
  CL->>E: caption_folder(…)
  E->>FS: scan → trabalho (pula .txt existentes, a menos de --overwrite)

  alt pasta inválida / sem imagens
    E-->>U: "ERROR…" / "No images …" → exit 1
  end

  E->>B: carregamento lazy (1º uso HF: baixa pesos · Ollama: precisa estar rodando com o modelo puxado)

  loop cada imagem (sequencial)
    E->>B: caption(imagem redimensionada [, contexto EXIF no Ollama])
    B-->>E: 1 texto por imagem (ou exceção)
    alt OK
      E->>FS: imagem.txt (UTF-8) ao lado da imagem — ou cópia em --output-dir
      E-->>U: stdout: barra + "Xs este arquivo" + ETA + hora prevista
    else FAIL
      E-->>U: stdout: linha FAIL com a exceção — continua
    end
  end

  E-->>CL: (written, skipped)
  CL-->>U: "Done. Wrote X captions, skipped Y." → exit 0
```

**Como ler:** mostra o que a versão Gradio esconde — a **camada de conveniência**
(`.bat` → `.venv` do ComfyUI → `normalize_path` contra aspas do Explorer) e os
**exit codes**. O loop é idêntico ao 1.2: mesma fachada-codificada, dois invokers.

---

## 2 · Descrição do sistema (síntese)

O inteiro está em [docs/01](docs/01-descricao-sistema.md); aqui, o essencial:

- **O que é:** ferramenta de lote **local** que percorre uma pasta de imagens e grava,
  para cada uma, uma legenda densa `.txt` sidecar — o formato exato que os treinadores de
  LoRA consomem. Ela **não treina**: para quando o dataset está pronto.
- **Nível da visão:** não é serviço — uma máquina, um operador, uma execução por vez,
  arquivos entrando e arquivos saindo. Duas fachadas (CLI `.bat` de drag & drop; UI
  Gradio em `127.0.0.1:7866`) sobre **um** motor (`caption_folder`).
- **Dentro do escopo:** 3 backends de visão (BLIP / Florence-2 in-process; Ollama via
  HTTP); prompt denso editável; contexto de arquivo/EXIF com whitelist (GPS sempre fora,
  opt-out); gatilho via prefix/suffix; resize; progresso com ETA; saída ao lado ou em
  pasta separada; conveniências Windows; workflows ComfyUI de referência.
- **Fora do escopo:** treino de LoRA; curadoria/revisão de legendas; nuem; empacotamento.
- **Integrações:** Ollama (runtime externo, 127.0.0.1:11434), Hugging Face Hub (pesos no
  1º uso), ComfyUI Desktop (ambiente Python, não chamada), filesystem (o dataset) e,
  indiretamente, os treinadores (`ai-toolkit`, `ltx-video-trainer`).
- **Restrições:** Windows-first; Pillow é a única dependência dura; GPU necessária
  (qwen3.6:27b ≈ 17 GB de VRAM); sem paralelismo; local-first por princípio.
- **Lacunas:** 10 registradas com impacto (`LG-01..10`), de “sem testes automatizados” a
  “rótulo da UI contradiz o código no resize”.

---

## 3 · Decisões e ajustes sobre o que a GenAI gerou

| | |
|---|---|
| Inferências **aprovadas** | 6 |
| Sugestões **modificadas** | 6 |
| Sugestões **descartadas** | 5 |

O detalhamento item a item está em [docs/02](docs/02-ia-diagramas.md); os destaques:

- **Acertos:** as 3 classes de backend nas suas duas naturezas de execução; 2 fachadas /
  1 motor (sem duplicar o fluxo); a semântica do ETA pelos **3 primeiros com sucesso**; a
  fronteira de privacidade do EXIF; a escolha da jornada crítica; a saída como contrato
  com os treinadores.
- **Invenções cortadas:** API REST + tabela `captions` + fila de jobs + Docker +
  autenticação + fallback em nuvem — a arquitetura canônica de “ferramenta web”
  transplantada para uma **ferramenta de lote** que não tem nenhum desses.
- **O caso que o modelo não pegou (só o código pega):** o rótulo da UI “Max side …
  **HF only**” contradiz o loop, que redimensiona **todos** os backends. O diagrama
  documenta o **comportamento do código**; o rótulo ficou registrado como lacuna real do
  sistema original (`LG-05`).

---

## 4 · Para agentes de desenvolvimento (o “a mais” da documentação)

A atividade pediu pensar neste repositório como **contexto futuro para agentes**. A
resposta está em [docs/03](docs/03-contractos-para-agentes.md): os documentos de
descritiva dão a *forma*, mas forma não é contrato — o doc 03 **trava com valor** as
~30 decisões que um agente inventaria (parâmetros, formato do `.txt`, whitelist de EXIF,
contratos dos backends, progresso/exit codes, restrições não-negociáveis,
**não-objetivos explícitos**) e deixa **5 decisões `ABERTO`** de propósito, com a regra:
agente **pergunta**, não chuta. O [AGENTS.md](AGENTS.md) é a porta de entrada disso.

---

## 5 · Como (re)renderizar os diagramas

Os blocos acima renderizam nativamente no GitHub a partir do **mesmo** conteúdo dos
`.mmd` de [`diagrams/`](diagrams/). Para PNGs fora do GitHub:

```bash
npx -y @mermaid-js/mermaid-cli -i diagrams/container-view.mmd -o diagrams/rendered/container-view.png
npx -y @mermaid-js/mermaid-cli -i diagrams/sequence-gradio-ollama.mmd -o diagrams/rendered/sequence-gradio-ollama.png
npx -y @mermaid-js/mermaid-cli -i diagrams/sequence-cli-batch.mmd -o diagrams/rendered/sequence-cli-batch.png
```

Os PNGs em [`diagrams/rendered/`](diagrams/rendered/) foram gerados com `mermaid-cli`
para uso offline; se divergirem dos blocos, **o bloco do README (e o `.mmd`) vencem**.

---

## Sugestão de leitura

1. [01 — Descrição do sistema](docs/01-descricao-sistema.md) — para formar impressão própria
2. [03 — Contratos para agentes](docs/03-contractos-para-agentes.md) — responde a pergunta da unidade
3. [02 — Uso de GenAI](docs/02-ia-diagramas.md) — o que o modelo acertou, corrigiu e inventou
4. [AGENTS.md](AGENTS.md) — como este repositório deve ser consumido por um agente

