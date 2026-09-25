# Handoff: sessão de 2026-09-25 (ACE-Step skills + Content Engine)

Contexto para retomar depois de um `/clear`. Leia isto primeiro.

## Repositórios e estado

| Repo | Branch | Estado |
|---|---|---|
| `SrLoula/acestep-skills` (privado) | `main` | commits `76f44d5` (skills iniciais), `a8832d4` (pós-produção + prompting), `31d0194` (workflows do turbo) + este handoff |
| `SrLoula/AI-Content-Automation-Engine` | `claude/automation-engine-mds-updates-76wyz9` | commit `f5930a9`: fix do 500 em `api/routers/music.py` (upload/criação de pasta em subpasta). **Sem PR aberto.** |
| Upstream `hans1801/AI-Content-Automation-Engine` | `main` | nada novo além do `bce51ba` (já está no fork) |

## Decisões do usuário

- Foco das skills: **voltadas ao pipeline do Content Engine**.
- ACE-Step roda **via ComfyUI na máquina do usuário** (porta padrão 8188). O modelo usado é o **ACE-Step 1.5 turbo 2B**.
- Repo de skills: `acestep-skills`, privado (o usuário criou; a integração do Claude não tem permissão para criar repos).
- A sessão roda na nuvem e **não acessa a máquina do usuário**. Os `.md` "atualizados localmente" só ficam visíveis depois de dar push.

## O que existe no acestep-skills

```
.claude-plugin/{plugin,marketplace}.json   # /plugin marketplace add SrLoula/acestep-skills
skills/acestep-comfyui/   scripts/acestep_comfy.py  (check | generate | batch), reference.md,
                          examples/batch.example.json, examples/ab-turbo.example.json
skills/acestep-prompting/ SKILL.md (legenda em prosa, letra, parâmetros, modelo, exemplos)
skills/acestep-post/      scripts/post.py (analyze | master | loop | mix | refclip | stems)
skills/acestep-bg-music/  scripts/bg_music.py (list | generate | push), presets.json (7 presets em prosa)
docs/knowledge.md         base de conhecimento com fontes
docs/turbo-workflows.md   turbo 1.5: shift, schedulers, timbre, áudio-para-áudio, repaint, receitas
tests/fake_comfy.py       ComfyUI falso (FAKE_AUDIO=arquivo.mp3 serve áudio real) para testar sem GPU
```

Tudo usa só a biblioteca padrão do Python; o `ffmpeg` é necessário para a pós-produção.

### Funcionalidades do `acestep_comfy.py`
- Workflow nativo: `CheckpointLoaderSimple` (aio) ou `UNETLoader` + `DualCLIPLoader(type=ace)` + `VAELoader` (split) → `ModelSamplingAuraFlow(shift 3)` → `TextEncodeAceStepAudio1.5` → `ConditioningZeroOut` (negativo) → `EmptyAceStep1.5LatentAudio` → `KSampler` → `VAEDecodeAudio` → `SaveAudioMP3/Opus/SaveAudio`.
- Detecta o modelo sozinho (aio ou split). Padrões por família: turbo 8 passos/cfg 1, base 50/6, sft 50/7.
- Flags novas: `--scheduler` (simple, beta, linear_quadratic…), `--sampler`, `--reference-audio` (nós `LoadAudio` → `VAEEncodeAudio` → `ReferenceTimbreAudio`), `--src-audio --denoise` (áudio-para-áudio), `--no-lm`/`--lm` (o LM desliga sozinho quando há áudio de entrada).
- Validações: aviso quando a legenda passa de 512 caracteres, erro quando a letra passa de 4096, e checagem de tom, compasso, idioma e BPM.
- Grava um `.json` ao lado de cada áudio com todos os parâmetros, o seed e o `prompt_id`.

### Funcionalidades do `bg_music.py`
- Presets: `finance-mindset`, `finance-strategy`, `stickman-explainer`, `storytelling-suspense`, `lofi-chill`, `epic-documentary`, `tech-future`. Formatos: `shorts` (75 s) e `longs` (210 s).
- Pós-processamento automático: `master` com alvo `bed` (−16 LUFS / −1,5 dBTP) seguido de `loop` sem emenda (crossfade de 2 compassos).
- Envio ao engine: `--push api` (POST `/api/music/upload`, porta 8000) ou `--push dir`. `--folder` para subpasta (exige o fix `f5930a9`). O `AudioTool` do engine só sorteia arquivos da raiz de `bg-music/`.
- Repassa `--reference-audio` e `--scheduler`.

## Principais aprendizados (detalhes e fontes em knowledge.md e turbo-workflows.md)

1. As legendas oficiais são **prosa em inglês** (mediana de 223 caracteres, máximo recomendado de 512). BPM e tom **não** entram na legenda.
2. No ComfyUI, BPM, tom, compasso e duração são sempre fixos (não há "auto"). Faixas estáveis: BPM de 60 a 180, tons comuns, 4/4, durações de 30–60 s ou 2–4 min.
3. O `shift=3` + `simple` + 8 passos do ComfyUI reproduz **exatamente** os timesteps oficiais do turbo.
4. Variantes do turbo: públicas (shift1, shift3, continuous) e **internas** (`fix-inst-*` para instrumental e `rl`). O serviço oficial usa a `turbo-fix-inst-shift-dynamic`. Vale acompanhar se sair o release.
5. O scheduler `linear_quadratic` tem relatos de áudio mais limpo (issues #956 e #957). **Ainda não foi validado**, por isso existe o A/B.
6. Repaint, cover e extend não são nativos no ComfyUI. Alternativas: o pacote `starsFriday/ComfyUI-ACEStep` (voltado ao XL) ou a UI oficial.
7. Os modelos XL têm áudio melhor, mas há relato de melodia menos coerente. O CFG do sft/base no ComfyUI já teve bug (#12322, fechado).
8. **O vídeo final do Content Engine sai em cerca de −22 LUFS**: o `amix` padrão em `tools/video_editing/ffmpeg.py::add_background_music` divide as entradas e a narração cai 6 dB. A correção proposta (sem aplicar) é `amix=...:normalize=0` + `sidechaincompress` (ducking) + `loudnorm=I=-14:TP=-1` no final. O `post.py mix` implementa isso; no teste, a mesma entrada deu −14,1.

## Pendências e próximos passos

- [ ] **Validar no ComfyUI real do usuário**: `acestep_comfy.py check`. Os nomes dos modelos podem ser diferentes do padrão; nesse caso, ajuste com `--checkpoint`/`--unet`.
- [ ] Rodar `batch skills/acestep-comfyui/examples/ab-turbo.example.json`, colher a preferência do usuário e ajustar os padrões (scheduler, shift, passos).
- [ ] Validar no 1.5 o áudio-para-áudio (`--src-audio`) e a referência de timbre, que ainda são experimentais.
- [ ] **Oferecido e aguardando resposta:** aplicar no engine a correção de loudness do mix (seção 8 acima).
- [ ] **Oferecido e aguardando resposta:** abrir PR da branch `claude/automation-engine-mds-updates-76wyz9` (fix do music router) para o `main`.
- [ ] Engine: os `.rules/` estão desatualizados em relação à v2 (não citam `api/`, `frontend/` nem Docker, e referenciam `storage_csv`/`prompt_shorts`, que não existem mais). Nada foi alterado.
- [ ] Engine: a branch `claude/hardware-monitoring-kqkjfy` tem só o `TODO.md` (UI de uso de CPU/RAM/GPU/temperatura), sem implementação.
- [ ] Opcional: integrar os nós de repaint/extend (`ComfyUI-ACEStep`) e o Matchering (masterização por referência).

## Como testar sem GPU

```bash
FAKE_AUDIO=algum.mp3 python3 tests/fake_comfy.py --port 8189 &
export COMFYUI_URL=http://127.0.0.1:8189
python3 skills/acestep-comfyui/scripts/acestep_comfy.py check
python3 skills/acestep-comfyui/scripts/acestep_comfy.py batch skills/acestep-comfyui/examples/ab-turbo.example.json
python3 skills/acestep-bg-music/scripts/bg_music.py generate --preset lofi-chill --out /tmp/bg
python3 skills/acestep-post/scripts/post.py analyze /tmp/bg/*.mp3 --target bed
```

## Atualização: varredura do `SrLoula/AI-Content-Automation-Engine-v2` (2026-09-25)

O **v2 é o projeto ativo de verdade** (749 arquivos, HEAD `6207c1f` "Prepare project changes for new repository").
O repositório `AI-Content-Automation-Engine` (v1) está **congelado**. Leia primeiro o `MASTER_CONTEXT.md` (2396 linhas),
depois o `AGENT_CONTEXT.md` (5012 linhas). Vários agentes trabalham no repo: DeepSeek Harness, Codex e Claude Code.
Clone nesta sessão: `/home/user/ai-content-automation-engine-v2`.

### Como o v2 usa o ACE-Step (diferente do que as acestep-skills assumem)
- Caminho principal: **servidor oficial** `openrouter.openrouter_api_server` na porta **8003**, com
  `POST /v1/chat/completions` e `thinking: true` (LM ligado, obrigatório). Wrapper no app: `tools/audio_generation/ace_step_generator.py`.
  O antigo wrapper da porta 8002, sem LM, degradava após ~3 min e não deve voltar.
- O ComfyUI (porta 8188, `D:\Renan\IA\ComfyUI`) também tem `ace_step_1.5_turbo_aio.safetensors` e `acestep_v1.5_turbo.safetensors`.
  O lote do Rooftop Cyberpunk foi gerado lá com **16 passos**, FLAC e ~163–176 s.
- GPU com 16 GB: o **XL turbo não coube** junto com o LM (erro de KV cache) e foi apagado. O SFT estava planejado, mas não baixado.

### Aprendizados validados pelo usuário (skill `generating-instrumental-music-ace-step` do v2)
1. **Coerência cai depois de ~3 min**, mesmo com o LM. Use faixas de **160–185 s com duração aleatória** e crossfade na montagem.
   → As acestep-skills usam `longs = 210 s`: **precisa corrigir**.
2. **Um gênero principal + um modificador.** Empilhar gêneros faz o áudio colapsar em ruído.
3. **Ordem das tags: gênero → clima → instrumentos**, usando o vocabulário real de produção do gênero.
4. **Faixa estável: só `[Instrumental]`.** Marcadores de voz vazios (`[Verse]`, `[Chorus]`) podem gerar **silêncio**; múltiplas seções podem causar mudança brusca.
5. **O timbre acústico é ponto fraco do turbo** (piano solo, cordas e baixo acústico soam artificiais). Mascarar com tape/vinyl ajuda no jazz, mas não salva piano solo nem orquestra.
   → Presets das acestep-skills em risco: `finance-mindset` (felt piano), `stickman-explainer` (pizzicato, ukulele) e `epic-documentary` (orquestra).
6. Leads "frenéticos" (guitarra solo, arpejo de synth) foram rejeitados; sax, Rhodes/piano elétrico e ambiência pura foram aprovados.
7. Para relaxar: pentatônica maior / lídio, "consonant harmony, no dissonant intervals". É uma alavanca secundária.
8. **O v2 diverge do tutorial oficial**: o usuário confirmou ao vivo que colocar o BPM **também como texto nas tags** ("80 bpm") ajudou. As acestep-skills dizem para não fazer isso. Decidir com A/B.
9. Rooftop Cyberpunk: uma identidade sonora por lote (chill lo-fi downtempo, Rhodes, baixo arredondado, pads), BPM provisório de 74, curadoria por audição, `track_review.md` e só aprovar com o OK do usuário (skill `rooftop-cyberpunk-music-curation`).
10. O servidor oficial só devolve `audios[0]` mesmo com `batch_size > 1`. Para variações, faça N requisições mudando só o `seed`.

### Problemas do v1 que continuam no v2
- `tools/video_editing/ffmpeg.py::add_background_music` (fluxo narrado, ~linha 1103): `amix` padrão, sem `normalize=0`, e **nenhum `loudnorm` em `tools/` ou `flows/`**. O vídeo final sai em ~−22 LUFS.
  (O mix de SFX do Ambiente, ~linha 286, já usa `normalize=0` + `alimiter`.)
- `api/routers/music.py` linhas 50 e 58: mesmo bug de `relative_to` (500 no upload para subpasta). O fix `f5930a9` só existe no v1.

### Outros pontos do v2
- `MASTER_CONTEXT.md §1` diz "v2 sem remote git"; agora ele está no GitHub, então isso ficou desatualizado. O README ainda aponta para o clone do `hans1801`.
- Skills do v2: `automation-engine-status-analysis`, `debugging-ambient-audio-transitions`, `driving-comfyui-via-api`,
  `generating-instrumental-music-ace-step`, `generating-music-yue2` (CC BY-NC, não comercial), `removing-vocals-comfyui`,
  `rooftop-cyberpunk-music-curation`. Estão espelhadas em `.agents/skills` e `.claude/skills`; duas diferem entre os espelhos.
- Spec mais nova: `docs/superpowers/specs/2026-09-24-ambient-audio-tail-analyzer.md` (helper implementado, não integrado).

### Decisão pendente com o usuário
Como conciliar as acestep-skills com o v2: (a) incorporar os aprendizados do v2 nas acestep-skills e adicionar um
backend para o servidor oficial (porta 8003); (b) levar só o `post.py` e as correções de loudness para dentro do v2;
(c) as duas coisas.
