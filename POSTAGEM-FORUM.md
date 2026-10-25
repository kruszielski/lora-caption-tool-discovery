# Texto para a postagem no fórum

Duas versões do mesmo conteúdo: uma completa e uma curta, caso o fórum tenha limite de
caracteres.

---

## Versão completa

**Repositório:** https://github.com/kruszielski/lora-caption-tool-discovery

Escolhi documentar um sistema que eu realmente uso: o `lora-caption-tool`, a ferramenta
que **gera as legendas (`.txt`)** dos meus datasets de treino de LoRA, com modelos de
visão locais (BLIP, Florence-2 ou Ollama). Há um traço de metalinguagem que achei
interessante: é um sistema **construído com GenAI** (Unidade 5) sendo **documentado com
GenAI** (Unidade 3) — ou seja, a atividade testou exatamente o ciclo que o professor
descreveu: diagrama em código → entrada para desenvolvimento com IA.

Concentrei tudo num repositório público: descrição em linguagem natural
(escopo, nível, fronteiras, integrações, restrições, 10 lacunas), o diagrama estrutural
( visão de containers inspirada no C4) e **dois** diagramas de sequência (a jornada
crítica pela UI Grada + a jornada CLI de drag & drop), o registro das decisões sobre a
saída do modelo, um documento de **contratos para agentes** e um `AGENTS.md`.

**O que o modelo inferiu corretamente**

Com acesso ao código-fonte, ele acertou a *forma* da arquitetura e detalhes que eu teria
que caçar: os três backends nas suas **duas naturezas de execução** (HF in-process ×
Ollama externo via HTTP); as **duas fachadas sobre um único motor** (`caption_folder`) —
sem criar dois fluxos; a semântica exata do **ETA pela média dos 3 primeiros arquivos com
sucesso**; a fronteira de **privacidade do EXIF** (whitelist de 34 campos, GPS sempre
fora, opt-out como decisão do usuário); e escolheu a **jornada certa** — a execução de
lote é a única para a qual a ferramenta existe — com o fluxo real: scan → contagem
(skip de `.txt` existentes) → carregamento lazy → loop com erro isolado por arquivo.
Também descreveu a saída como **contrato com os treinadores** (`folder_path` +
`caption_ext: txt`), não como “arquivos de texto”.

**O que precisei ajustar**

O padrão clássico: o modelo **quer transformar pipeline de lote em serviço**. A primeira
redação veio com API REST, tabela `captions`, fila com workers, Docker e fallback em
nuvem — tudo **cortado** (D-01..05): não há nenhum desses no sistema. Também corrigi
Ollama de “biblioteca” para **runtime externo** (o adaptador HTTP é que está na
fronteira), o ComfyUI de “integração” para **ambiente Python** (nenhuma chamada existe),
e subi o diagrama de componente para **container**. O caso mais interessante foi o
modelo **não** pegar: o rótulo da UI diz “Max side … HF only”, mas o código aplica o
resize a **todos** os backends. O código venceu; registrei como lacuna real do sistema
original (LG-05) e o diagrama documenta o comportamento do código, não o do rótulo.

**O que a documentação precisaria ter a mais para um agente construir o sistema**

Forma (escopo/fronteiras/jornadas) não é contrato. O documento `docs/03` lista as ~30
decisões que um agente seria forçado a **inventar** se não estivessem travadas, e trava
cada uma: parâmetros com tipo/padrão/regra, formato exato do `.txt` (UTF-8,
`prefix+legenda+suffix`+`\n`), whitelist de EXIF com exclusão de GPS, contrato dos três
backends (incluindo timeout 300 s e a **ausência de retry**), contrato de progresso e
exit codes, restrições não-negociáveis (local-first, sem banco, sem nuvem) e
**não-objetivos explícitos** (é aí que a GenAI “completa” o sistema sozinha). Cinco
decisões ficaram marcadas `ABERTO` de propósito — retry, relatório de falhas, padrão de
backend entre as fachadas, semântica de gif, preview de legenda — com a regra: o agente
**pergunta**, não chuta. O `AGENTS.md` no repositório já aplica a lição da Unidade 2:
*qualquer número sem marcador é lacuna, nunca valor*.

---

## Versão curta

**Repositório:** https://github.com/kruszielski/lora-caption-tool-discovery

Documentei um sistema real meu (`lora-caption-tool`, que gera legendas `.txt` para
datasets de LoRA) com descrição em linguagem natural (incluindo 10 lacunas), diagrama de
containers (C4) e **duas** sequências em Mermaid — todas geradas por GenAI **com acesso
ao código** e verificadas linha a linha.

**Acertos do modelo:** os 3 backends nas duas naturezas (HF in-process × Ollama HTTP),
2 fachadas / 1 motor, ETA pelos 3 primeiros com sucesso, privacidade de EXIF e a escolha
da jornada crítica (o lote).

**Ajustes:** cortei o serviço web + banco + fila + Docker + nuvem que ele propôs (o
sistema é lote local); corrigi Ollama para runtime externo e ComfyUI para ambiente
Python; subi o diagrama para o nível de container. O caso que ele **não** pegou: o
rótulo da UI “Max side … HF only” contradiz o código (resize vale para todos) — o código
venceu e vira lacuna documentada (LG-05).

**Para o agente construir sem inventar:** `docs/03` trava as ~30 decisões com valor
(contratos de parâmetro, `.txt`, EXIF, backends, progresso, exit codes), declara
não-objetivos explícitos (é onde a GenAI “completa” sozinha) e deixa 5 decisões
`ABERTO` para o agente **perguntar**. O repositório traz um `AGENTS.md` com a regra da
Unidade 2: número sem marcador é lacuna, nunca valor.
