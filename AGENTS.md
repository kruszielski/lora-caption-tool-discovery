# AGENTS.md — contexto para agentes de desenvolvimento

Se você é um agente de IA recebendo **este repositório** (ou o código upstream) para
**implementar ou modificar o `lora-caption-tool`**, leia este arquivo primeiro. Ele existe
para fazer você **parar de inventar decisões**.

## O que o sistema é

Ferramenta **local, de uso único, de lote** que gera legendas `.txt` (modelos de visão:
BLIP / Florence-2 / Ollama) para datasets de treino de LoRA. Duas fachadas (CLI e Gradio)
sobre **um** motor (`caption_folder`). Saída: pasta consumível por `ai-toolkit` /
`ltx-video-trainer` / ComfyUI. **Não é um serviço e não treina.**

## Onde está cada coisa

- **Código upstream:** <https://github.com/kruszielski/lora-caption-tool>
- **Documentação (este repositório):**
  `docs/01` (descrição em linguagem natural) · `docs/02` (revisão da GenAI: o que foi
  aproveitado/modificado/descartado) · `docs/03` (**contratos — ler antes de escrever
  código**) · `diagrams/` (arquitetura em Mermaid, fonte de verdade).

## Regras

1. **`docs/03` é o contrato.** Replicar os valores `DECIDIDO`. Para qualquer item
   `ABERTO`, **perguntar ao dono do sistema** — não inventar número, padrão, política de
   retry ou comportamento de erro.
2. **Não adicionar:** banco de dados, fila, workers, API REST, Docker, autenticação,
   multiusuário, fallback em nuvem, empacotamento. (O registro de descarte da Unidade 3 —
   `docs/02`, D-01..05 — mostra que essa é o modo de falha documentado da GenAI neste
   sistema.)
3. **Local-first e privacidade são princípios de design:** nenhuma chamada externa além do
   Ollama local e do download de pesos HF no primeiro uso; EXIF-GPS nunca sai da máquina;
   o opt-out de metadados é de primeira classe.
4. **Conveniência Windows-first** (`.bat` de drag & drop), mas o núcleo permanece
   agnóstico de SO.
5. **Os `.mmd` de `diagrams/` são a arquitetura.** Se você mudar um componente, uma
   fronteira ou uma mensagem de uma jornada, muda o diagrama **no mesmo commit**
   (diagrams-as-code: o diagrama é código — versionado, revisado em PR, re-renderizado).
6. **O sistema não tem testes automatizados** (LG-01). Ao adicionar comportamento,
   adicione o teste mínimo correspondente — os helpers puros (`clean_path_string`,
   `normalize_path`, `list_images`, `resize`, `_fmt_exif_value`, `_eta_note`) são os mais
   baratos de cobrir.

## O que não confiar

Rótulos da UI e trechos do README upstream **onde conflitam com o código**. Caso já
documentado: “Max side … HF only” — o resize se aplica a **todos** os backends (LG-05).
**O código vence; este repositório documenta qual é o comportamento pretendido.**
