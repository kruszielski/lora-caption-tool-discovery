# 03 — O que esta documentação precisaria ter para um agente construir o sistema

A pergunta da Unidade 3: *o que a documentação precisaria ter a mais para um agente
construir o sistema sem inventar decisões?*

Os documentos 01 e 02 dão ao agente a **forma** (escopo, fronteiras, integrações,
jornadas). Mas forma não é contrato: cada uma das decisões abaixo, se deixada em branco,
seria **inventada** pelo agente — e invenção é o modo de falha documentado na Unidade 2 e
reproduzido aqui (D-01..05).

Por isso este documento faz duas coisas: (a) lista as decisões que um agente seria forçado
a inventar se não estivessem aqui; e (b) **trava cada uma com valor**, marcado como
`DECIDIDO` (rastreável no código atual — o agente deve replicar) ou `ABERTO` (lacuna real:
o agente deve **perguntar**, não chutar).

---

## 1. Contrato de parâmetros (fachadas)

| Parâmetro | Tipo | Padrão | Regra | Status |
|---|---|---|---|---|
| `input_dir` | path | obrigatório | normalizar (tirar aspas externas, expandir `~`); precisa ser diretório existente → senão `ERROR` + parada | `DECIDIDO` |
| `output_dir` | path \| nenhum | nenhum | `nenhum` → `.txt` ao lado de cada imagem; definido → **copia** a imagem + `.txt` espelhando a estrutura relativa | `DECIDIDO` |
| `prefix` / `suffix` | str | `""` | legenda final = `f"{prefix}{legenda}{suffix}".strip()`; caso de uso: gatilho `"mychar, "` | `DECIDIDO` |
| `model` | enum | CLI: `blip-large`; UI: `ollama` | `blip-large` \| `blip-base` \| `florence2-base` \| `ollama` — **as fachadas discordam no padrão** (ver ABERTO-3) | `DECIDIDO` + `ABERTO` |
| `recursive` | bool | `false` | `true` → `rglob` em extensões maiúsculas e minúsculas; `false` → só `iterdir` | `DECIDIDO` |
| `overwrite` | bool | `false` | `false` e `.txt` existe → pula (conta como `skipped`); `true` → regera | `DECIDIDO` |
| `max_side` | int | `1024` | `0` = sem resize; senão escala a borda maior (LANCZOS) — **aplica a todos os backends** (o rótulo da UI “HF only” é dívida, LG-05) | `DECIDIDO` |
| `device` | str \| nenhum | auto: `cuda` se disponível, senão `cpu` | só backends HF | `DECIDIDO` |
| `ollama_host` | url | `http://127.0.0.1:11434` | — | `DECIDIDO` |
| `ollama_model` | str | `qwen3.6:27b` | precisa estar puxado; `URLError` → mensagem com dica de `ollama pull` | `DECIDIDO` |
| `ollama_extra` | str | `""` | anexado ao prompt como bloco “Additional instructions” | `DECIDIDO` |
| `ollama_prompt` | str \| nenhum | prompt denso embutido | `--ollama-prompt-file` (UTF-8) ou textbox da UI; vazio/nenhum → embutido | `DECIDIDO` |
| `ollama_file_context` | bool | **`true`** | `false` → sem bloco de basename/EXIF (opt-out de privacidade) | `DECIDIDO` |

---

## 2. Contrato de arquivo (saída)

- Um `.txt` por imagem, **mesmo nome-base**, extensão da imagem trocada por `.txt`.
- Codificação **UTF-8**; conteúdo = a linha da legenda (`prefix+legenda+suffix` já
  concatenados e `strip()`ados) + `"\n"`.
- Com `output_dir`: árvore espelhada (só as imagens que vão ser legendadas são copiadas).
- `.txt` existente **nunca** é tocado sem `overwrite` — sem backup, sem diff, sem versão.

---

## 3. Contrato de varredura

- `EXT = {.png, .jpg, .jpeg, .webp, .bmp, .gif}`; `gif` é legendado como **estático**
  (primeiro quadro via Pillow) — semântica a revisar (LG-06, ABERTO-4).
---

## 4. Contrato de backends

| | BLIP (large/base) | Florence-2 base | Ollama |
|---|---|---|---|
| Execução | in-process `transformers` | in-process `transformers` (`trust_remote_code`) | **externa** via HTTP `POST /api/chat` |
| Pesos | `Salesforce/blip-image-captioning-{large,base}` | `microsoft/Florence-2-base` | puxado pelo usuário (padrão `qwen3.6:27b`, ~17 GB) |
| Inferência | `float16` se cuda, senão `float32`; `generate(max_length=120, min_length=5, num_beams=5)` | tarefa `<DETAILED_CAPTION>`; `max_new_tokens=256, num_beams=3, do_sample=False`; `post_process_generation` | 1 mensagem `role=user`; conteúdo = prompt + contexto + extra; imagem = **JPEG q90 base64** da imagem já redimensionada; `stream:false`; **timeout 300 s** |
| Falha | exceção → linha `FAIL` do arquivo (loop continua) | idem | `URLError` → “start Ollama / pull model”; `HTTPError` → código + corpo; `content` vazio → erro |
| **Retry** | — | — | **ABERTO-1 (LG-02)**: inexistente hoje; o agente deve **perguntar** (quantas vezes? backoff? falha longa distorce o ETA?) |

---

## 5. Contrato de contexto de arquivo (Ollama)

- Bloco: `Basename:` + `Name without extension:` + cabeçalho + 1 linha por campo EXIF da whitelist.
- **Whitelist (34 campos):** Make, Model, Software, Artist, Copyright, DateTime,
  DateTimeOriginal, DateTimeDigitized, ImageDescription, UniqueCameraModel, HostComputer,
  LensMake, LensModel, LensSerialNumber, FocalLength, FocalLengthIn35mmFilm, FNumber,
  ExposureTime, ISOSpeedRatings, ISOSpeed, ExposureProgram, WhiteBalance, MeteringMode,
  LightSource, Flash, CameraOwnerName, BodySerialNumber, CameraSerialNumber, XPTitle,
  XPAuthor, XPComment, XPSubject.
- **GPS sempre excluído** (tag `0x8825` e IFD `GPSInfo`); truncamento: bytes em 200
  chars, strings em 300.
- Instrução ao modelo: usar o metadado **só para confirmar** (câmera, lente, artista)
  **quando consistente com os pixels; em divergência, a imagem vence**.
- Sem EXIF legível → linha explícita “(none or not read — e.g. some PNGs/webp, or
  stripped files)”.

---

## 6. Contrato de progresso e saída

- 1 linha por arquivo: `[barra 20] feito/total | Xs este arquivo | ETA … | OK|FAIL rel`.
- **ETA = restantes × média dos 3 primeiros arquivos com sucesso**; com menos de 3:
  “(provisional, n/3 times)”; mais **horário previsto** de término.
- Encerramento: `Done. Wrote {ok} captions, skipped {sk}.`
- **Exit codes (CLI):** `1` em pasta inválida / sem imagens / prompt-file ilegível;
  `0` nos demais — **inclusive com arquivos FAIL** (ABERTO-2, LG-03: deve falha parcial
  dar exit != 0 e relatório final de falhas?).
- O Gradio reporta o mesmo progresso via `gr.Progress` + caixa de log (o mesmo `log_print`).

---

## 7. Restrições não-funcionais (intocáveis pelo agente)

1. **Local-first:** nenhuma API externa, nenhum telemetria, nenhuma chave.
2. **Sem banco, sem estado** além dos arquivos do filesystem.
3. **Uma execução por vez**; processo único; arquivos sequenciais.
4. **Pillow é a única dependência dura** (Gradio e transformers são opcionais por backend).
5. **Conveniência Windows-first** (`.bat` de drag & drop), mas o núcleo deve permanecer
   agnóstico de SO.
6. **Privacidade:** o “EXIF desligado” é opção de primeira classe, não patch.

---

## 8. Não-objetivos explícitos (para travar a “completude” do agente)

Não treinar. Não curar/revisar legendas. Não paralelizar. Não chamar nuvem. Não
multiusuário. Não empacotar (é pasta de scripts). Não registrar nó no ComfyUI.
*(Cada item corresponde a algo que um modelo propôs e foi descartado — docs/02, D-01..05.)*

---

## 9. O que continua `ABERTO` (o agente deve **perguntar**, não chutar)

| # | Decisão aberta | Lacuna |
|---|---|---|
| ABERTO-1 | Política de retry/timeout do Ollama (quantas tentativas? backoff? efeito no ETA?) | LG-02 |
| ABERTO-2 | Relatório final de falhas + exit code em falha parcial | LG-03 |
| ABERTO-3 | Padrão de `model` entre as fachadas (CLI `blip-large` × UI `ollama`) — unificar? | — |
| ABERTO-4 | Semântica de `gif` (estático × quadro representativo × multi-legenda?) | LG-06 |
| ABERTO-5 | Preview/revisão de legenda (se entra, é **nova funcionalidade** — fora do as-is) | LG-04 |

---

> **Regra adotada (da Unidade 2, mantida aqui):** qualquer número, padrão ou política
> **sem marcador** `DECIDIDO` neste documento é **lacuna a registrar** — nunca valor a
> adotar. Um agente que receber este repositório deve tratar as 5 decisões `ABERTO` como
> **perguntas ao dono do sistema**, e os diagramas de `diagrams/` como a arquitetura
> contra a qual toda mudança será cobrada.

- Ordem **ordenada** (determinística), maiúsculas e minúsculas aceitas.
- Sem imagens → mensagem `No images (extensions: …)` + parada (CLI: exit 1).
- Todas com `.txt` já existente (sem `overwrite`) → `No captions to write — all .txt
  sidecars exist. Use --overwrite to regenerate.`
