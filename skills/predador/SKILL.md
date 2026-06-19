---
name: predador
description: >-
  Orquestrador de inteligência competitiva de VSL para macOS. A partir de UMA URL de página
  de vendas (player vTurb/ConverteAI), monta um DOSSIÊ COMPLETO numa pasta única do Swipe File:
  (1) baixa a VSL em MP4, (2) extrai o áudio em MP3 e transcreve com whisper (.txt/.srt), (3) roda
  a espionagem de funil, (4) disseca a arquitetura da VSL em 14 camadas, e (5) gera um índice
  HTML (dossie.html) juntando tudo. É a skill MAESTRO: NÃO refaz o trabalho das outras, ela
  invoca as sub-skills `espionagem-nobrega` e `filemon-dissector` e os scripts do app
  "MP4 para MP3". Use SEMPRE que o usuário colar uma URL de VSL/página de vendas e disser
  "predador", "/predador", "monta o dossiê", "espiona essa VSL", "dossiê completo dessa VSL",
  "raio-x completo". Roda LOCALMENTE no Mac (precisa do app, ffmpeg e whisper instalados).
metadata:
  author: spyzeer
  version: 1.0.0
  category: competitive-intelligence
  tags: [vsl, espionagem, swipe-file, vturb, transcricao, dossie]
---

# 🐆 PREDADOR — Dossiê de VSL em um comando

Você é o **maestro**. A partir de **uma URL**, você produz uma **pasta-dossiê** completa no Swipe
File com a VSL baixada, transcrita, espionada e dissecada — e um índice HTML que amarra tudo.
Você **não reescreve** o que as sub-skills fazem: você as **chama na ordem certa**, salva cada
entregável na mesma pasta e gera o índice final.

## Configuração (caminhos fixos da máquina)

```
APP_RES   = /Users/olipeoliver/Developer/vsl/MP4 para MP3.app/Contents/Resources
SWIPE_BASE = /Users/olipeoliver/Library/CloudStorage/GoogleDrive-spyzeer@gmail.com/Meu Drive/DR. SOCIETY - NEW LIFE/SWIPE FILE
```

Os dois scripts ficam em `APP_RES`: `baixar-vsl.sh` (URL → MP4) e `convert.sh` (MP4 → MP3 + `.txt`/`.srt`).
**Sempre** entre aspas em tudo — os caminhos têm espaços e acentos.

## Entrada

O usuário fornece **uma URL** da página da VSL (ex.: `/predador https://site.com/oferta/`).
Flag opcional `--sem-video` → pula download/transcrição/dissecação e roda só a espionagem.

---

## Passo a passo

### Passo 0 — Preparar a pasta-dossiê

Rode no Bash (substitua `<URL>` pela URL recebida):

```bash
APP_RES="/Users/olipeoliver/Developer/vsl/MP4 para MP3.app/Contents/Resources"
SWIPE_BASE="/Users/olipeoliver/Library/CloudStorage/GoogleDrive-spyzeer@gmail.com/Meu Drive/DR. SOCIETY - NEW LIFE/SWIPE FILE"
URL="<URL>"

SLUG=$(printf '%s' "$URL" | awk -F/ '{for(i=NF;i>0;i--) if($i!=""){print $i; exit}}' | sed 's/[^A-Za-z0-9._-]/_/g')
[ -z "$SLUG" ] && SLUG="vsl"
HOST=$(printf '%s' "$URL" | awk -F/ '{print $3}' | sed 's/[^A-Za-z0-9.-]/_/g')
DEST="$SWIPE_BASE/$(date +%Y-%m-%d)_${HOST}_${SLUG}"
mkdir -p "$DEST"
echo "DEST=$DEST"
```

Guarde o valor de `DEST` — é a pasta onde **tudo** será salvo.

### Passo 1 — Baixar a VSL (rápido, primeiro plano)

```bash
MP4="$("$APP_RES/baixar-vsl.sh" "$URL" "$DEST" 2>"$DEST/_download.log")"
echo "MP4=$MP4"
```

- Se sair **vazio** ou com erro (ver `_download.log`), a página pode não ter VSL vTurb. **Não aborte
  o dossiê**: marque `SEM_VIDEO=1`, siga para a espionagem (Passo 3) e registre a falha no índice.
- Se deu certo, `MP4` é o caminho do `.mp4` dentro de `DEST`.

### Passo 2 — Transcrever em SEGUNDO PLANO (a etapa mais lenta)

O whisper large-v3 numa VSL de ~40-50 min demora vários minutos. **Dispare em background** e siga
trabalhando na espionagem enquanto transcreve:

```bash
"$APP_RES/convert.sh" "$MP4" > "$DEST/_transcricao.log" 2>&1
```

Rode esse comando com **run_in_background = true**. Comportamento exato do `convert.sh` (confirmado):
cria uma subpasta `<stem>/` ao lado do mp4 e coloca lá `<stem>.mp3`, `<stem>.txt`, `<stem>.srt`,
**MOVE o `<stem>.mp4` pra dentro dela também**, e gera um extra `<stem> - INFO.txt` (métricas:
duração, palavras, ritmo de fala). Idioma fixo `pt`, VAD ativo. Você recolhe tudo no Passo 4.

### Passo 3 — Espionagem do funil (em paralelo com a transcrição)

Enquanto o Passo 2 roda, **invoque a sub-skill `espionagem-nobrega`** passando a `URL`.
Quando ela entregar o relatório HTML, **salve em `"$DEST/espionagem.html"`** (se a skill já salvar
em arquivo, mova/renomeie para esse caminho). Extraia e anote os achados-chave (stack: player/
checkout, Offer IDs, preços; testes A/B; ângulo; reputação) — você vai usá-los no resumo do índice.

### Passo 4 — Esperar a transcrição e achatar a pasta

Garanta que o job em background do Passo 2 terminou (o `.txt` precisa existir **dentro da subpasta**
`<stem>/`). O `convert.sh` coloca tudo nessa subpasta (incluindo o mp4, que ele MOVE pra lá, e um
`<stem> - INFO.txt`). Suba os entregáveis pra raiz do dossiê e identifique cada um:

```bash
# sobe mp4/mp3/txt/srt (e o INFO) da subpasta pra raiz do dossiê, depois apaga a subpasta vazia
find "$DEST" -mindepth 2 -type f \
  \( -name '*.mp4' -o -name '*.mp3' -o -name '*.txt' -o -name '*.srt' \) \
  -exec mv -f {} "$DEST"/ \;
find "$DEST" -mindepth 1 -type d -empty -delete

# a TRANSCRIÇÃO é o .txt que NÃO termina em "INFO.txt" (esse outro é só métricas)
TXT="$(find "$DEST" -maxdepth 1 -type f -name '*.txt' ! -iname '*info.txt' | head -1)"
MP4="$(find "$DEST" -maxdepth 1 -type f -name '*.mp4' | head -1)"
MP3="$(find "$DEST" -maxdepth 1 -type f -name '*.mp3' | head -1)"
SRT="$(find "$DEST" -maxdepth 1 -type f -name '*.srt' | head -1)"
INFO="$(find "$DEST" -maxdepth 1 -type f -iname '*info.txt' | head -1)"
echo "TXT=$TXT"; echo "MP4=$MP4"; echo "MP3=$MP3"; echo "SRT=$SRT"; echo "INFO=$INFO"
```

⚠️ **Nunca** passe o `"<stem> - INFO.txt"` para a `filemon-dissector` — a transcrição é o `$TXT` acima.
O `$INFO` traz duração/ritmo de fala prontos: use-os no resumo executivo do índice.

### Passo 5 — Dissecar a arquitetura (filemon-dissector)

Se houver transcrição (`TXT` não vazio), **invoque a sub-skill `filemon-dissector`** fornecendo o
**conteúdo de `transcricao.txt`**. Salve o entregável em `"$DEST/dissecacao-filemon"` — `.html` se a
saída for HTML, ou `.md` se for texto/markdown. Anote o resumo (ângulo, hook+apelido, promessa,
mecanismo, oferta) para o índice.

### Passo 6 — Gerar o índice `dossie.html`

Crie `"$DEST/dossie.html"` — um índice **self-contained**, estética **preto-e-branco "ScalenX"**
(coerente com as outras skills). Use o esqueleto abaixo, preenchendo os `{{...}}`. Como `dossie.html`
fica na **mesma pasta** que os arquivos, cada link é só o **basename** (ex.: `{{MP4}}` →
`basename "$MP4"`, `{{TXT}}` → `basename "$TXT"`, `{{DISSEC}}` → nome do arquivo da dissecação).
Preencha o **resumo executivo** com o que a espionagem e a dissecação revelaram (stack, preço,
ângulo, mecanismo, oferta, A/B) — e puxe duração/ritmo de fala do `$INFO` (Passo 4).

```html
<!doctype html><html lang="pt-BR"><head><meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>Dossiê — {{TITULO}}</title>
<style>
  :root{--bg:#0a0a0a;--fg:#f5f5f5;--mut:#9a9a9a;--line:#262626;--accent:#fff}
  *{box-sizing:border-box}body{margin:0;background:var(--bg);color:var(--fg);
    font:16px/1.6 -apple-system,Inter,Segoe UI,Roboto,sans-serif;padding:48px 24px}
  .wrap{max-width:960px;margin:0 auto}h1{font-size:28px;letter-spacing:-.02em;margin:0 0 4px}
  .meta{color:var(--mut);font-size:14px;margin-bottom:32px;word-break:break-all}
  .grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(220px,1fr));gap:14px;margin:24px 0}
  a.card{display:block;padding:18px;border:1px solid var(--line);border-radius:12px;
    text-decoration:none;color:var(--fg);transition:.15s}a.card:hover{border-color:var(--accent);transform:translateY(-2px)}
  .card .k{font-size:12px;color:var(--mut);text-transform:uppercase;letter-spacing:.08em}
  .card .v{font-size:17px;margin-top:6px}.sec{border-top:1px solid var(--line);margin-top:32px;padding-top:24px}
  h2{font-size:13px;text-transform:uppercase;letter-spacing:.1em;color:var(--mut)}
  ul{padding-left:18px}code{background:#1a1a1a;padding:1px 6px;border-radius:5px}
</style></head><body><div class="wrap">
  <h1>🐆 {{TITULO}}</h1>
  <div class="meta">{{URL}} · {{DATA}}</div>
  <div class="grid">
    <a class="card" href="{{MP4}}"><div class="k">Vídeo</div><div class="v">▶ VSL (MP4)</div></a>
    <a class="card" href="{{MP3}}"><div class="k">Áudio</div><div class="v">♪ MP3</div></a>
    <a class="card" href="{{TXT}}"><div class="k">Transcrição</div><div class="v">📄 .txt / .srt</div></a>
    <a class="card" href="espionagem.html"><div class="k">Espionagem</div><div class="v">🔎 Funil & Stack</div></a>
    <a class="card" href="{{DISSEC}}"><div class="k">Dissecação</div><div class="v">🧬 14 camadas</div></a>
  </div>
  <div class="sec"><h2>Resumo executivo</h2>
    <ul>
      <li><b>Stack:</b> {{STACK}}</li>
      <li><b>Preço / Oferta:</b> {{PRECO}}</li>
      <li><b>Ângulo / Hook:</b> {{ANGULO}}</li>
      <li><b>Mecanismo:</b> {{MECANISMO}}</li>
      <li><b>Testes A/B:</b> {{ABTEST}}</li>
    </ul>
  </div>
</div></body></html>
```

### Passo 7 — Abrir a pasta e reportar

```bash
open "$DEST"
```

Encerre com uma mensagem curta ao usuário: caminho do dossiê, o que ficou pronto (✅) e o que
falhou (⚠️), e 2-3 destaques do resumo executivo.

---

## Paralelismo (a sacada de tempo)

A transcrição (whisper) é de longe a etapa mais demorada. Por isso a ordem é:
**baixar (rápido) → disparar transcrição em background → fazer a espionagem enquanto isso →
esperar a transcrição → dissecar → montar índice.** Assim a espionagem "roda de graça" durante a
transcrição.

## Regras de robustez

- **Sempre** entre aspas em `$DEST`, `$MP4`, `$APP_RES`, `$SWIPE_BASE` (têm espaços/acentos).
- Download falhou → não derrube o dossiê: faça a espionagem mesmo assim e marque ⚠️ no índice.
- Nunca sobrescreva um dossiê anterior: a pasta leva data + host no nome.
- Transcrição ≠ métricas: o `convert.sh` gera DOIS `.txt` (`<stem>.txt` = transcrição e
  `<stem> - INFO.txt` = métricas). Para a dissecação, use SEMPRE o que **não** termina em `INFO.txt`.
- Se `espionagem-nobrega` ou `filemon-dissector` devolverem texto em vez de arquivo, **você** salva
  o conteúdo na pasta com a extensão certa.
- Tudo — mp4, mp3, txt, srt, espionagem.html, dissecacao, dossie.html — fica **dentro de `$DEST`**.

## Extensões futuras (não implementar agora)

`--sem-video` (só espionagem) · rodar mais skills filemon (raio-x, oferta, mecanismo) sobre a mesma
transcrição · gerar um swipe consolidado de vários concorrentes.
