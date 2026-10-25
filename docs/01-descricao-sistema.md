# 01 — Descrição do Sistema em Linguagem Natural

> **Objeto:** [`lora-caption-tool`](https://github.com/kruszielski/lora-caption-tool) —
> ferramenta real do aluno para gerar legendas de datasets de treino de LoRA.
>
> **Método:** redigida com apoio de GenAI **que teve acesso ao código-fonte**
> (`src/`, `scripts/`, `workflows/`, `requirements.txt`, README upstream) e depois
> verificada linha a linha contra ele. Cada afirmação deste documento rastreia para algo
> que existe no repositório — ou está marcada como lacuna.

---

## 1. O que é e por que existe

O `lora-caption-tool` é uma **ferramenta de lote local** que percorre uma pasta de imagens
e, para cada imagem, gera uma **legenda densa em um arquivo `.txt` sidecar** — exatamente o
formato que os treinadores de LoRA consomem. Ela existe porque escrever legendas à mão é
lento e inconsistente: num dataset de LoRA (dezenas a centenas de imagens), legenda com
vocabulário e nível de detalhe variando de arquivo para arquivo vira **ruído** para o
treinamento.

O valor entregue é **o dataset pronto**: após uma execução, a pasta é consumível diretamente
por `ai-toolkit`, `ltx-video-trainer` ou pelos workflows de ComfyUI. A ferramenta **não
treina** — ela para quando o dataset está pronto.

---

## 2. Nível da visão

O sistema **não é um serviço**. Não há servidor permanente (a UI Gradio é opcional, local,
e escuta em `127.0.0.1:7866`), não há banco de dados, não há usuários, não há fila, não há
multiusuário. É **uma máquina, um operador, uma execução por vez, arquivos entrando e
arquivos saindo**.

A visão correta é a de um **pipeline de desktop com duas fachadas sobre um único motor**:

1. **Fachada CLI** — `src/caption_dataset_for_lora.py` executado em console, ou via `.bat`
   com a pasta arrastada em cima (fluxo primário no Windows);
2. **Fachada web (opcional)** — `src/caption_gradio_app.py` (Gradio, porta 7866), com um
   formulário que espelha os parâmetros da CLI.

Ambas as fachadas chamam a **mesma função** — `caption_folder(...)` — que concentra todo o
comportamento. A UI é *reembalagem* da CLI, não uma segunda implementação.

---

## 3. Escopo

### 3.1 Dentro do sistema

- Percorrer pasta procurando `png, jpg, jpeg, webp, bmp, gif` (com `--recursive`, que
  consulta extensões em maiúsculas e minúsculas);
- **Três backends de visão**, escolhidos em tempo de execução, de duas naturezas distintas:
  - `blip-large` / `blip-base` — **in-process**, via `transformers` (Hugging Face);
  - `florence2-base` — **in-process** (`transformers`, tarefa `<DETAILED_CAPTION>`);
  - `ollama` — **runtime externo** local (padrão `qwen3.6:27b`) via HTTP `POST /api/chat` —
    o único backend sem dependência Python (`urllib` da stdlib);
- **Prompt editável**: instrução densa embutida (estilo, cena, POV, composição, luz,
  pessoas, meio, sinais foto × pintura × desenho), substituída por `--ollama-prompt-file`
  (arquivo UTF-8), pelo textbox da UI ou por `ollama_prompt=` em chamadas de biblioteca;
- **Contexto de arquivo** (apenas Ollama): basename + nome sem extensão + EXIF
  **whitelist** (34 campos: câmera, lente, Artist, Copyright, XP…) anexado ao prompt;
  **GPS é sempre excluído**; desligável via `--ollama-no-file-context` / checkbox;
- **Palavra-gatilho** via `--prefix` / `--suffix` (ex.: `"mychar, "`);
- **Saída**: `imagem.txt` ao lado da imagem, ou cópia das imagens + `.txt` em
  `--output-dir` (espelhando a estrutura relativa); `.txt` existente é pulado, a menos de
  `--overwrite`;
- **Barra de progresso ASCII** com **ETA pela média dos 3 primeiros arquivos com sucesso**
  (+ marcação “provisional” com menos de 3) e **horário previsto de término**;
- **Resize** da borda maior até `--max-side` (padrão 1024; `0` = desligado) **antes de
  enviar a imagem a qualquer backend**;
- UI Gradio (127.0.0.1:7866) com os mesmos parâmetros, caixa de log e barra de progresso;
- **Conveniências Windows**: `.bat` de arrastar-pasta, `.bat`/`.ps1` de atalho na área de
  trabalho, `.bat` que instala o Gradio na `.venv` do ComfyUI;
- **Dois workflows de exemplo** (JSON ComfyUI): um grafo utilitário de processamento de
  dataset em lote (`VHS_LoadImagesPath` + strings + `SaveImageKJ` + `Note`) e um
  referencial de treinamento (apenas `Note`s apontando para `ltx-video-trainer`,
  `ai-toolkit` e o artigo do Hugging Face).

### 3.2 Fora do sistema (o que a ferramenta **não** faz)

- **Treino de LoRA** — pertence a `ai-toolkit` / `ltx-video-trainer` / ComfyUI;
- **Curadoria e revisão** — não há fila de aprovar/rejeitar legendas nem preview da legenda
  junto à imagem (a UI mostra apenas log de texto);
- **Nuvem** — nenhuma API externa, nenhum upload, nenhuma chave: local-first é princípio de
  design;
- **Empacotamento** — não há `pyproject.toml`, PyPI ou instalador: é uma pasta de
  scripts + `requirements.txt` + `.gitignore`.

---

## 4. Fronteiras e responsabilidades

| Componente | Responsabilidade | Fronteira (o que **não** faz) |
|---|---|---|
| `src/caption_dataset_for_lora.py` | Núcleo: `list_images`, `resize`, EXIF, backends, loop `caption_folder`, CLI `argparse` | Não treina, não cura, não persiste estado além dos arquivos |
| `src/caption_gradio_app.py` | Apresentação: formulário, progresso, log; mapeia UI → `caption_folder` | Não contém regra de negócio (tudo delegado ao núcleo) |
| `scripts/*.bat` / `.ps1` | Lançamento: drag & drop, atalho de área de trabalho, instalar Gradio na `.venv` do ComfyUI | Não processa imagem |
| `workflows/*.json` | Documentação/referência do pipeline *downstream* no ComfyUI | Não são carregados nem interpretados pela ferramenta |
| `docs/prompt_template.example.txt` | Exemplo canônico do prompt denso para `--ollama-prompt-file` | — |

**Onde a fronteira passa:** o sistema começa quando existe uma pasta de imagens no disco
local e termina quando os `.txt` são gravados. Tudo *upstream* (captura/criação/curadoria)
e tudo *downstream* (treino, inferência, publicação) está fora — mas a documentação precisa
descrever essas interfaces, porque **o formato do dataset é contrato** com os treinadores.

---

## 5. Integrações

| Com | Como | O que cruza a fronteira |
|---|---|---|
| **Ollama** (runtime externo, `http://127.0.0.1:11434`) | `POST /api/chat`, `stream:false`, 1 request por imagem (JPEG q90, base64, já redimensionado), timeout 300 s | Prompt (+ contexto) + imagem → texto da legenda. O modelo precisa estar puxado (`ollama pull qwen3.6:27b`, ~17 GB) |
| **Hugging Face Hub** | `transformers` baixa pesos no **primeiro uso** (BLIP ~1,5 GB; Florence-2 ~1,2 GB) | Só pesos; a inferência em seguida é 100 % local |
| **ComfyUI Desktop** | **Não é chamada** — é *ambiente*: os `.bat` assumem o repositório ao lado da `.venv` do ComfyUI e reutilizam-na (evitando o `python.exe` do Miniconda) | Ambiente Python |
| **Sistema de arquivos** | Lê imagens + EXIF; escreve `.txt` (UTF-8, `prefix+legenda+suffix` + `\n`) ao lado ou em `--output-dir` | O dataset |
| **Treinadores** (`ai-toolkit`, `ltx-video-trainer`, ComfyUI) | **Indireta**: a pasta de saída é a entrada deles | `folder_path` + `caption_ext: txt` — o formato LoRA |

---

## 6. Restrições

- **Plataforma:** Windows-first — caminhos com aspas vindos do Explorer, `.bat`, `.ps1`,
  `pause`, `.venv` específico do ComfyUI Desktop. O código é Python puro e roda em outros
  SOs, mas **nenhuma conveniência existe** para eles.
- **Dependências:** `Pillow` é o único requisito duro; `gradio` é opcional (UI);
  `transformers` + `torch` são opcionais (só para BLIP/Florence). O backend Ollama é só stdlib.
- **Hardware:** modelos de visão locais exigem GPU — `qwen3.6:27b` é ~17 GB de VRAM
  (cabe em RTX 3090 24 GB); alternativas mais leves existem (`qwen3.5:9b`, `qwen2.5vl:7b`).
  Os modelos HF baixam gigabytes no primeiro uso.
- **Concorrência:** um processo, arquivos sequenciais, uma execução por vez. Não há
  paralelismo nem retomada: “retomada” é apenas implícita (pasta com `.txt` já existentes
  produz `No captions to write — use --overwrite`).
- **Privacidade:** EXIF só sai para o modelo se o usuário deixar ligado, só da whitelist,
  e **sem GPS** — restrição deliberada, porque metadados de foto de comissão/stock podem
  conter dados que o autor não quer expor a uma IA.
- **Qualidade da saída:** depende do modelo de visão; o prompt manda “confie nos pixels”,
  mas nada valida o resultado — o próprio README recomenda **revisão manual** de amostra
  antes de treinar.

---

## 7. Lacunas (registro explícito)

Numeradas no estilo do registro de lacunas da Unidade 2; cada item tem impacto em quem
consumir esta documentação — em especial, um **agente de desenvolvimento**:

| # | Lacuna | Impacto |
|---|---|---|
| LG-01 | **Sem testes automatizados** no repositório (apenas `py_compile` como fumaça) | Um agente que reconstruísse o sistema não tem oráculo de aceitação; o contrato comportamental precisa estar documentado (doc 03) |
| LG-02 | **Sem política de retry** no Ollama além do timeout único de 300 s: arquivo que falha permanece falho até reexecução manual | Comportamento do lote sob falha parcial é indefinido; ETA é distorcido por falhas longas |
| LG-03 | **Sem relatório de falhas**: as falhas ficam só no log; não há lista final de “o que falhou e por quê”, e o exit code é 0 mesmo com falhas | O operador precisa reler o log inteiro para saber o que refazer |
| LG-04 | **Sem preview/revisão de legenda** na UI (só log de texto) | Controle de qualidade fica fora do sistema (leitura manual dos `.txt`) |
| LG-05 | **Rótulo da UI “Max side … HF only” contradiz o código** — o loop aplica `resize()` a toda imagem antes de chamar o backend, **inclusive o Ollama** | Inconsistência interface × código que um modelo lendo só a UI reproduziria; o código vence (doc 02, M-04) |
| LG-06 | **`gif` é aceito, mas decodificado como imagem estática** (primeiro quadro via Pillow) | Dataset animado recebe legenda do quadro 1; semântica não documentada |
| LG-07 | **O workflow ComfyUI de batch não é executável pela ferramenta** — depende de nós que o usuário precisa ter instalado (KJ Nodes, VHS) | O “pronto para uso” da pasta `workflows/` é apenas referencial |
| LG-08 | **Versões não fixadas** (`transformers>=4.40`, `gradio>=4.0,<6`): o comportamento do 1º uso depende do que HF/Ollama servirem no dia | Reprodutibilidade do “primeiro uso” não garantida |
| LG-09 | **Prompt único e duplicado**: a instrução densa vive embutida no código *e* em `docs/prompt_template.example.txt`; edição por dataset é manual (arquivo/textbox) | Não há registro de prompts; as duas cópias podem divergir |
| LG-10 | **Ollama sem streaming** (`stream:false`): nenhum feedback durante a geração de uma única legenda (27b: 30–60 s/imagem) | A UI parece “congelada” entre linhas de log; só o ETA por arquivo explica o que acontece |
