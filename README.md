# yt-pubx â€” publique qualquer vÃ­deo no YouTube com um comando

[![yt-pubx](guia/assets/banner.jpg)](https://inematds.github.io/yt-pubx/guia/)

**ðŸ‡§ðŸ‡· [PortuguÃªs](README.md) Â· ðŸ‡ºðŸ‡¸ [English](README.en.md) Â· ðŸ‡ªðŸ‡¸ [EspaÃ±ol](README.es.md)**

## O que Ã©

O yt-pubx Ã© um programa de linha de comando que publica vÃ­deos no YouTube por vocÃª. Ele serve para quem publica com frequÃªncia, em um ou vÃ¡rios canais, e nÃ£o quer preencher tÃ­tulo, descriÃ§Ã£o, tags e thumb Ã  mÃ£o toda vez. A partir de um texto curto ou da legenda do vÃ­deo, o Codex CLI escreve os textos e cria a arte da thumb, e o yt-pubx envia tudo pela API oficial do YouTube. Para usar, vocÃª precisa de Python, do Codex CLI logado e de um projeto gratuito no Google Cloud com a YouTube Data API ativada.

## ðŸ“– Guia de uso

Guia completo (landing + passo a passo): **https://inematds.github.io/yt-pubx/guia/**

---

VocÃª aponta um vÃ­deo (arquivo ou link) e o `yt-pubx` faz o resto:

1. **Escreve tÃ­tulo, descriÃ§Ã£o e tags** com o Codex CLI, a partir de um contexto seu e/ou da legenda.
2. **Cria a thumb**: a arte sai do gerador de imagem do Codex (`image_gen`) e o `yt-pubx`
   aplica por cima a frase, a marca e as cores do seu canal.
3. **Publica direto** no canal escolhido (pÃºblico, nÃ£o listado, privado ou agendado),
   aplica a thumb e, se houver SRT, envia a legenda.

Funciona com quantos canais vocÃª quiser. Tudo roda na sua mÃ¡quina; o texto e a arte
usam a sua assinatura do Codex (ChatGPT), sem chave de API de modelo.

```bash
yt-pubx publicar aula.mp4 --contexto "Aula 3 do curso X. PÃ¡gina: https://..." --legenda aula.srt
```

---

## 1. O que vocÃª precisa

| Item | Para quÃª |
|---|---|
| Python 3.10+ com `requests` e `Pillow` | rodar o script (`pip install requests pillow`) |
| `ffprobe` (vem com o ffmpeg) | ler a duraÃ§Ã£o do vÃ­deo |
| [Codex CLI](https://github.com/openai/codex) logado (`npm i -g @openai/codex` â†’ `codex login`) | escrever o texto e gerar a arte da thumb |
| Um projeto no Google Cloud com a **YouTube Data API v3** | publicar (passo 3) |
| `yt-dlp` (opcional) | publicar a partir de link do YouTube, TikTok, Instagramâ€¦ |

## 2. Instalar

```bash
git clone https://github.com/inematds/yt-pubx.git
cd yt-pubx
ln -s "$PWD/yt-pubx" ~/.local/bin/yt-pubx   # opcional: chamar de qualquer pasta
yt-pubx init                                 # cria ~/.config/yt-pubx/config.json
```

## 3. Liberar o YouTube no Google Cloud (uma vez)

1. Entre em <https://console.cloud.google.com/>, crie um projeto (ex.: `meu-yt-pubx`).
2. **APIs e serviÃ§os â†’ Biblioteca** â†’ procure **YouTube Data API v3** â†’ **Ativar**.
3. **APIs e serviÃ§os â†’ Tela de consentimento OAuth** (Google Auth Platform):
   - Tipo de usuÃ¡rio **Externo**; nome do app e seu e-mail.
   - Em **PÃºblico-alvo / UsuÃ¡rios de teste**, adicione o e-mail da conta Google dona do canal.
4. **APIs e serviÃ§os â†’ Credenciais â†’ Criar credenciais â†’ ID do cliente OAuth**:
   - Tipo de aplicativo: **App para computador** (Desktop). Se vocÃª jÃ¡ tem um client
     **Aplicativo da Web**, ele tambÃ©m serve: cadastre nele um endereÃ§o `http://localhost:<porta>/...`
     em *URIs de redirecionamento autorizados* (detalhes no guia abaixo).
   - Baixe o JSON (`client_secret_....json`). Guarde fora de qualquer repositÃ³rio.

> **Primeira vez?** Leia [docs/autorizar-canal.md](docs/autorizar-canal.md): o passo a passo
> sem termos tÃ©cnicos, com cada erro que o Google mostra e o que fazer.

> **Duas armadilhas do Google que valem saber antes:**
> - **App em modo "Teste"**: o acesso expira em **7 dias** e vocÃª precisa rodar `yt-pubx auth` de novo.
>   Para nÃ£o expirar, clique em **Publicar app** (passa para "Em produÃ§Ã£o"; para uso prÃ³prio, a tela
>   de "app nÃ£o verificado" aparece, Ã© sÃ³ seguir em *AvanÃ§ado â†’ continuar*).
> - **Projeto sem auditoria**: o YouTube pode travar como **privado** os vÃ­deos enviados por projetos
>   de API novos que ainda nÃ£o passaram pela auditoria dos serviÃ§os de API do YouTube. Se os seus
>   vÃ­deos ficarem presos em privado, peÃ§a a auditoria no formulÃ¡rio de *YouTube API Services*
>   (Ã© gratuito) ou publique como `private` e troque para pÃºblico no Studio.
> - **Cota**: a cota padrÃ£o Ã© de 10.000 unidades/dia; cada envio de vÃ­deo consome a maior parte
>   disso (o suficiente para poucos vÃ­deos por dia). DÃ¡ para pedir aumento no console.
> - **Thumb personalizada** exige canal verificado por telefone (<https://www.youtube.com/verify>).

## 4. Configurar o canal

Edite `~/.config/yt-pubx/config.json`:

```json
{
  "canal_padrao": "principal",
  "codex_modelo": "",
  "baixador": "yt-dlp -f \"bv*[ext=mp4]+ba[ext=m4a]/b[ext=mp4]/b\" -o \"{outdir}/%(title).80s.%(ext)s\" {url}",
  "imagem_reserva": { "url": "", "modelo": "" },
  "canais": {
    "principal": {
      "nome": "Meu Canal",
      "url": "https://www.youtube.com/@meucanal",
      "sobre": "Canal de tutoriais curtos sobre fotografia com celular.",
      "assinatura": "Inscreva-se: https://www.youtube.com/@meucanal",
      "idioma": "pt",
      "categoria": "27",
      "privacy": "public",
      "thumb": {
        "marca": "MEU CANAL",
        "fonte": "",
        "cor_texto": "#ffffff",
        "cor_marca": "#f0e805",
        "cor_destaque": "#f91f06"
      }
    }
  }
}
```

| Campo | O que faz |
|---|---|
| `sobre` | uma frase sobre o canal; o Codex usa para acertar o tom |
| `assinatura` | linha fixa que fecha toda descriÃ§Ã£o (link do site, inscriÃ§Ã£oâ€¦) |
| `categoria` | id da categoria do YouTube (27 EducaÃ§Ã£o, 28 CiÃªncia e tecnologia, 22 Pessoas e blogs) |
| `thumb.marca` | texto da etiqueta no canto superior esquerdo (vazio = sem etiqueta) |
| `thumb.fonte` | caminho de um `.ttf` (vazio = Montserrat ExtraBold se instalada, senÃ£o DejaVu Sans Bold) |
| `codex_modelo` | modelo do Codex (vazio = o padrÃ£o da sua conta) |
| `baixador` | comando para baixar links que nÃ£o sÃ£o `.mp4` direto; `{url}` e `{outdir}` sÃ£o trocados |
| `imagem_reserva` | opcional: servidor de imagem local usado se o Codex falhar (`POST {model, prompt, width, height}` â†’ `{"image": base64}`) |

Depois autorize (abre o navegador; entre com a conta dona do canal):

```bash
yt-pubx auth principal --client-secret ~/Downloads/client_secret_XXXX.json
yt-pubx canais
```

Navegador em outra mÃ¡quina (servidor, acesso remoto)? Depois do **Permitir** a pÃ¡gina dÃ¡ erro de
conexÃ£o: copie a URL da barra de endereÃ§o, cole no terminal do `yt-pubx auth` e tecle Enter.
O link de autorizaÃ§Ã£o tambÃ©m fica em `~/.config/yt-pubx/auth-<canal>.txt`, para copiar sem quebra.

O token fica em `~/.config/yt-pubx/tokens/principal.json` (permissÃ£o 600). Para mais canais,
repita: um bloco em `canais` + um `yt-pubx auth <nome>`. O mesmo `client_secret` serve para
todos os canais; cada `auth` Ã© feito logado na conta dona daquele canal (ou escolhendo o canal
de marca na tela do Google).

> JÃ¡ tem as credenciais guardadas em outro lugar? Em vez do `auth`, coloque no canal
> `"credenciais_cmd": "<comando>"`: um comando que imprime
> `{"client_id": "...", "client_secret": "...", "refresh_token": "..."}`.

## 5. Publicar

```bash
# conferir sem enviar: gera texto e thumb e mostra tudo
yt-pubx publicar video.mp4 --contexto "Do que o vÃ­deo trata + links oficiais" --dry-run

# gostou? publique exatamente o que viu (sem gerar outra arte)
yt-pubx publicar video.mp4 --plano ~/.local/share/yt-pubx/trabalhos/<pasta>/plano.json

# trocar sÃ³ a frase da thumb
yt-pubx publicar video.mp4 --plano .../plano.json --frase-thumb "Do zero ao ar"

# outro canal, a partir de um link, agendado
yt-pubx publicar https://exemplo.com/video.mp4 --canal segundo --contexto "..." --agendar "2026-12-01 18:00"

# tudo Ã  mÃ£o, sem Codex
yt-pubx publicar video.mp4 --title "..." --description "..." --tags "a,b,c" --thumb capa.jpg
```

| OpÃ§Ã£o | |
|---|---|
| `--contexto`, `--contexto-arquivo`, `--legenda` | matÃ©ria-prima para o Codex escrever (a legenda SRT tambÃ©m Ã© enviada) |
| `--title`, `--description`, `--tags` | fixa o que vocÃª jÃ¡ tem; o Codex completa o resto |
| `--frase-thumb`, `--cena-thumb` | texto da thumb (2â€“4 palavras) e a cena da arte |
| `--thumb capa.jpg` / `--thumb-arte arte.png` | thumb pronta / arte pronta (sÃ³ aplica frase e marca) |
| `--sem-thumb` | nÃ£o gera nem envia thumb (automÃ¡tico em vÃ­deo vertical/Short) |
| `--privacy public\|unlisted\|private`, `--agendar "AAAA-MM-DD HH:MM"` | visibilidade; agendar sobe privado e o YouTube publica no horÃ¡rio |
| `--canal`, `--idioma`, `--categoria` | sobrepÃµem o config |
| `--sem-legenda` | usa o SRT sÃ³ como contexto |
| `--dry-run`, `--plano`, `--manter` | conferir antes; publicar o que foi conferido; guardar o vÃ­deo baixado |

VÃ­deo vertical curto (atÃ© 3 min) vira **Short** automaticamente; ponha `#Shorts` no tÃ­tulo se quiser
deixar explÃ­cito.

**SaÃ­das:** `~/.local/share/yt-pubx/trabalhos/<data>-<nome>/` (plano.json, thumb.jpg, thumb_arte.png)
e `~/.local/share/yt-pubx/historico.jsonl` (uma linha por publicaÃ§Ã£o).

**Editar um vÃ­deo jÃ¡ publicado** (tÃ­tulo, descriÃ§Ã£o ou tags; o resto fica como estÃ¡):

```bash
./yt-pubx atualizar https://www.youtube.com/watch?v=XXXXXXXXXXX --canal lives1 --description-arquivo descricao.txt
```

## 6. Usar com um agente (Claude Code, Codex)

Ponha numa instruÃ§Ã£o do seu agente (CLAUDE.md / AGENTS.md):

> Quando eu pedir para publicar um vÃ­deo no YouTube: junte o contexto (assunto, links oficiais,
> SRT), rode `yt-pubx publicar <video> --contexto "..." --dry-run`, abra a `thumb.jpg` e confira
> se o texto estÃ¡ legÃ­vel e se tÃ­tulo e descriÃ§Ã£o estÃ£o corretos; entÃ£o publique com
> `--plano <pasta>/plano.json` e me passe o link.

Foi assim que o yt-pubx nasceu: o pedido "publica esses 3 vÃ­deos" virou trÃªs dry-runs conferidos,
uma troca de frase na thumb e trÃªs publicaÃ§Ãµes com thumb em menos de 5 minutos.

## Problemas comuns

| Mensagem | Causa / saÃ­da |
|---|---|
| `redirect_uri_mismatch` | client "Aplicativo da Web" sem o endereÃ§o local cadastrado â†’ [docs/autorizar-canal.md](docs/autorizar-canal.md) |
| `single value: access_type` | link copiado quebrado do terminal â†’ abra pelo `~/.config/yt-pubx/auth-<canal>.txt` |
| erro de conexÃ£o em `localhost` depois do Permitir | navegador em outra mÃ¡quina â†’ cole a URL da barra no terminal do `auth` |
| `token OAuth recusado` | token expirado (app em modo Teste: 7 dias) â†’ `yt-pubx auth <canal>` |
| `thumb nÃ£o aplicada â€¦ verify` | canal sem verificaÃ§Ã£o por telefone |
| `legenda nÃ£o enviada â€¦ force-ssl` | token antigo sem o escopo de legendas â†’ `yt-pubx auth` de novo |
| `upload nÃ£o iniciou (403) quotaExceeded` | cota do dia acabou â†’ amanhÃ£, ou peÃ§a aumento |
| vÃ­deo publicado fica privado | projeto de API sem auditoria (ver passo 3) |
| `codex nÃ£o gerou arte.png` | Codex sem gerador de imagem na sua conta â†’ `--thumb-arte`, `--thumb` ou `imagem_reserva` |

## LicenÃ§a

MIT. Projeto educacional independente; nÃ£o Ã© produto do Google, do YouTube nem da OpenAI.

## Modelo e esforÃ§o por etapa

Use Python 3.11+ com `pip install -r requirements.txt` (requests, Pillow e PyYAML). ConfiguraÃ§Ã£o comentada em `~/.config/yt-pubx/ia.yaml`, exemplo em `ia.exemplo.yaml`; comandos em [IA.md](IA.md). Metadados configurÃ¡veis, imagem Codex pela assinatura, traduÃ§Ã£o no inema-traduz.

## DistribuiÃ§Ã£o apÃ³s upload

Encaminhamento opcional por canal, com estado e retomada sem novo upload: [configuraÃ§Ã£o e limites](docs/pos-publicacao.md).

---

Projeto adaptado da comunidade INEMA TDS - Marca Arckhe