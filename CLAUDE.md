# Simulador de Previsibilidade Imobiliária — 4 Improvements

Contexto completo do projeto para qualquer sessão de Claude Code (local ou web).
Lê este ficheiro antes de alterar o que quer que seja.

## O que é este projeto

Funil de captação de leads da **4 Improvements** (agência portuguesa de performance,
CRM, IA e operação comercial para o setor imobiliário). O lead responde a um
simulador de 6 perguntas, recebe um diagnóstico personalizado por cluster de
maturidade comercial e marca uma sessão estratégica gratuita de 30 minutos.

- **Público-alvo:** consultores imobiliários, team leaders, diretores comerciais
  e brokers/donos de agência em Portugal.
- **Tráfego:** campanhas Meta Ads focadas em dores (follow-up, previsibilidade,
  velocidade de resposta, base de dados parada, controlo de equipa).
- **CEO:** Pedro Ferreira. Pode haver um segundo closer, Eike Paulino. Por isso
  a página de obrigado NÃO menciona nomes: diz sempre "a nossa equipa".

## Regras de escrita (obrigatórias)

- **Português europeu (PT-PT)**: telemóvel, e-mail, faturação, equipa, contactar,
  "está a perder", angariações. Nunca PT-BR.
- **NUNCA usar travessão** (— ou –) em nenhum texto visível ao utilizador nem em
  copys. Substituir por vírgula, dois pontos ou ponto final. Regra explícita do Luan.
- Tratamento por "você" implícito nas páginas do simulador; copys de anúncios de
  vídeo podem usar "tu" (mais natural falado).

## Arquitetura — 3 páginas estáticas

Stack: HTML + CSS + JS vanilla, zero dependências além de Google Fonts e da
YouTube IFrame API. Deploy em GitHub Pages.

```
index.html      Simulador: hero + 6 perguntas + 3 passos de contacto
   |  (sessionStorage + webhook + pixel)  ->  redirect
resultado.html  Diagnóstico por cluster + retoma opcional do vídeo + calendário
   |  (webhook session_booked + pixel)    ->  redirect
obrigado.html   Confirmação da sessão: data/hora, próximos passos, Google Calendar
```

- URL live: `https://4improvementsportugal-ops.github.io/simulador-previsibilidade/`
- Repo: `github.com/4improvementsportugal-ops/simulador-previsibilidade` (branch `main`)
- As 3 páginas têm meta-tags anti-cache (Cache-Control/Pragma/Expires) para o
  GitHub Pages nunca servir versões velhas.

### Contrato do sessionStorage (chave `4i_simulador`)

Escrito pelo `index.html` no submit; lido por `resultado.html` e `obrigado.html`:

```json
{
  "formData": { "perfil", "estado_operacao", "velocidade_resposta",
                "ia_base_dados", "faturacao_anual", "objetivo_90d",
                "nome", "empresa", "email", "telefone" },
  "score": 0-12, "score_max": 12,
  "cluster": "reativa|potencial|semprev|pronta", "cluster_label": "...",
  "foco_label": "...",
  "vsl": { "started": bool, "ended": bool, "time": segundos },
  "sessao_data": "YYYY-MM-DD",  // acrescentado pelo resultado ao confirmar
  "sessao_hora": "HH:MM",       // idem
  "timestamp": "ISO"
}
```

Guards de acesso direto: `resultado.html` sem `formData` redireciona para
`index.html`; `obrigado.html` sem `sessao_data`/`sessao_hora` idem (com
`throw` para parar a execução do script durante o redirect).

## Funil do index.html (6 perguntas + 3 contactos = 9 passos)

| # | Campo | Conteúdo | Score |
|---|---|---|---|
| 1 | `perfil` | Consultor / Team Leader / Diretor comercial / Broker | 0 |
| 2 | `estado_operacao` | 5 frases de auto-descrição (reativo → maduro) | 0–4 |
| 3 | `velocidade_resposta` | <30s / minutos / 1h / horas / dia seguinte | 4–0 |
| 4 | `ia_base_dados` | IA + cadências a reativar BD → nada | 4–0 |
| 5 | `faturacao_anual` | 80–100k / 100–300k / +300k euros | 0 (qualificação pura) |
| 6 | `objetivo_90d` | O "sonho": velocidade/followup/recuperar_bd/angariacoes/previsibilidade/escalar | 0 |
| 7 | contacto | nome + empresa (2 inputs no mesmo passo) | — |
| 8 | contacto | email | — |
| 9 | contacto | telemóvel → submit | — |

- Transições emocionais entre passos 1, 2, 3 e 5 (objeto `transitions`, 1.7s).
- Loading falso de 3.2s com 4 sub-passos animados após a pergunta 6
  (`advanceFromStep` quando `stepNum === TOTAL_QUALI`).
- `TOTAL_QUALI = 6`, `TOTAL_STEPS = 9`. Se adicionares perguntas, atualiza:
  ids `step-N`, labels "Pergunta X de Y", `data-next`, os `if (step === N)` do
  `validateInputStep`, o handler de Enter, as constantes e o `mapLabel`/payload.

### Validação do telemóvel (sem OTP)

Só aceita telemóvel português: limpa espaços/hífens/prefixos (+351, 00351),
exige `/^9[1236]\d{7}$/` (91/92/93/96) e rejeita sequências de dígitos todos
iguais. Normaliza para `+351XXXXXXXXX` antes de enviar.

**Histórico:** foi implementado OTP obrigatório por SMS via Firebase Phone Auth
(commit `e643a8a`) e revertido a pedido do Luan (commit `a9467c6`) para evitar
custo/fricção por agora. Se voltar a ser pedido, esse commit tem a implementação
completa (passo 9 de código, reCAPTCHA invisível, fail-open). Alternativa soft
discutida: workflow GHL que envia WhatsApp automático e marca leads sem entrega.

## Scoring e clusters (0–12, das perguntas 2+3+4)

| Score | Slug | Nome |
|---|---|---|
| 0–3 | `reativa` | Operação Reativa |
| 4–6 | `potencial` | Potencial Mal Aproveitado |
| 7–9 | `semprev` | Operação Sem Previsibilidade |
| 10–12 | `pronta` | Operação Pronta para Escalar |

No `resultado.html`, cada cluster tem um resumo (`clusterDefs`) e 3 ações
prioritárias (`getCriticalPoints`) que variam consoante o perfil: broker /
diretor / team_leader recebem versão "equipa"; consultor recebe versão pessoal.
A ação que corresponde ao `objetivo_90d` escolhido é promovida para o topo
(técnica de mirror do sonho do lead). A caixa "O seu foco para os próximos 90
dias" devolve o sonho literalmente (`dreamLabels`).

## Vídeo VSL (YouTube ID `Y7diCaqebrY`, canal 4Improvements)

Decisão em vigor (mudou ao longo do projeto, esta é a atual):

- **index.html**: widget de vídeo SEMPRE visível desde o primeiro ecrã.
  - Desktop (≥1024px): painel FIXO à direita a ocupar ~42vw, verticalmente
    centrado; o formulário recentra-se na metade esquerda via
    `body.fv-docked{padding-right:46vw}`. Sem arrasto no desktop.
  - Mobile (<1024px): janela flutuante pequena, arrastável por pointer events.
  - **Sem botão de fechar nem de minimizar** (removidos a pedido). A barra do
    título mostra só "Mensagem do CEO · 4 Improvements".
  - **Player cativo**: `controls:0, fs:0, disablekb:1, rel:0` + um DIV escudo
    transparente (`#fvShield`, z-index acima do iframe) que bloqueia todos os
    cliques no player nativo. É impossível abrir o youtube.com. O clique no
    vídeo alterna apenas play/pausa (ícone de play reaparece em pausa).
  - Usa a YouTube IFrame API para saber `getCurrentTime()`; o estado
    (`started/ended/time`) vai no sessionStorage via `window.__fvState()`.
- **resultado.html**: o bloco de vídeo está `hidden` por defeito. Só aparece
  se `vsl.started && !vsl.ended` (pessoa ficou a meio no index), com badge
  "Continuar a ver" e retoma no segundo gravado menos 3 (playerVar `start`).
  Ninguém vê o vídeo duas vezes. Player igualmente cativo com escudo
  (`#ytShield`). Botões de velocidade 1x/1.25x/1.5x/2x funcionam via
  `setPlaybackRate`.

**Histórico:** originalmente o vídeo era gate obrigatório no resultado
(anti-skip, diagnóstico bloqueado até ao ENDED). O Luan mudou para vídeo
opcional visível desde o início; o gate foi removido. O Pedro tinha pedido o
gate: se voltar atrás, ver commits `98518c1`/`9d942b1` (gate + anti-skip) e
`4544391` (remoção).

## Tracking — Meta Pixel `1506988540335331`

Estratégia: o evento **Lead só dispara quando a marcação se confirma**
(obrigado.html), para o Meta otimizar por bookings reais e não por formulários.

| Página | Evento Meta | Quando |
|---|---|---|
| index | `PageView` | load |
| index | `VSLPlayFloating` (custom) | play no widget |
| index | `VSLCompleted` (custom) | vídeo terminou no widget |
| index | `SubmitApplication` | submit do funil (intenção, NÃO é Lead) |
| resultado | `PageView` + `ViewContent` | load / diagnóstico renderizado |
| resultado | `VSLCompleted` (custom) | retoma terminada |
| resultado | `Schedule` | clique em "Confirmar sessão estratégica" |
| obrigado | `PageView` + **`Lead`** (com eventID p/ dedup CAPI) | chegada = booking confirmado |

GA4 (`gtag`) espelha com `page_view`, `schedule`, `generate_lead`,
`vsl_completed` — só dispara se o gtag existir na página (guards `typeof`).

**No Meta Ads Manager: configurar `Lead` como evento de conversão.**

## Webhook GoHighLevel (LeadConnector)

URL (igual nas 3 páginas que o usam):
`https://services.leadconnectorhq.com/hooks/lwdT8imgib43lDC9X1SC/webhook-trigger/6kZS8vcwprpSbH3o4rP4`

Dois POSTs (JSON, `keepalive:true` para sobreviver aos redirects):

1. **Lead** (index, no submit): `nome, empresa, email, telemovel, perfil,
   perfil_slug, estado_operacao, velocidade_resposta, ia_base_dados,
   faturacao_anual, faturacao_anual_slug, foco_90d, foco_90d_slug,
   score_maturidade, score_max, cluster_slug, cluster_label, origem,
   pagina_url, timestamp, raw{...}`.
2. **Marcação** (resultado, ao confirmar): tudo acima + `sessao_data`,
   `sessao_hora`, `event_type: "session_booked"`. No GHL distingue-se pelo
   campo `event_type`.

## Calendário de marcação

O calendário no `resultado.html` é **vanilla/cosmético**: gera o mês, desativa
fins-de-semana e datas passadas, slots fixos 09/10/11/14/15/16/17h. NÃO
verifica disponibilidade real nem bloqueia slots entre leads diferentes.

**Plano em curso:** substituí-lo pelo calendário nativo do GoHighLevel.
Já foi criado no GHL um calendário **Round Robin** chamado "Simulador
Previsibilidade" com `Maximum bookings per slot (per user) = 1`. Falta:
adicionar Pedro + Eike em Staff & location, definir Availability, configurar
redirect pós-marcação para `obrigado.html`, e obter o embed code para eu
trocar no `resultado.html`. Google Calendar dos anfitriões ainda não está
ligado (decidiu-se avançar sem, o GHL bloqueia slots na mesma).

Na página de obrigado, o botão "Adicionar ao Google Calendar" gera um link
`calendar.google.com/render?action=TEMPLATE` com data/hora da marcação
(+30 min), título "Sessão Estratégica · 4 Improvements".

## Contactos e assets

- WhatsApp e telefone: **+351 963 934 488** (`wa.me/351963934488`) em todas as
  páginas (bloco alternativo + CTAs fixos mobile).
- Logo (CDN GHL, usado como brand bar + favicon):
  `https://assets.cdn.filesafe.space/lwdT8imgib43lDC9X1SC/media/69c1cdbe3e56b9aeea9f211b.png`
- Vídeo VSL no YouTube: `Y7diCaqebrY`. Há também os ficheiros originais numa
  pasta Drive ("EDITADOS", `video-vsl.mp4` id `17tmkYdQE89f001Pv1bMFJaBf5AiwYNgT`).

## Design system

- Fundo página `#080B1F` com radial-gradients indigo/cyan/violet + grelha subtil.
- Barra de marca BRANCA a toda a largura do ecrã (`.brand-wrap` fora do
  contentor central, nas 3 páginas). Cartão principal escuro `.card-shell`
  (max-width 720px, radius 24px) centrado.
- Gradiente primário: `#22D3EE → #6366F1 → #8B5CF6` (botões pill, progress,
  seleções). WhatsApp `#25D366`. Sucesso/lime `#34D399`.
- Fontes: Space Grotesk (display), Inter (corpo), JetBrains Mono (labels,
  contadores, tempos).
- Opções do funil: pills full-width com check circular à direita; auto-avanço
  ao clicar (sem botão continuar nos passos de escolha).
- Mobile-first; inputs com font-size ≥16px (evita zoom iOS); CTAs fixos no
  rodapé só em <768px.

## Fluxo de trabalho git

- Trabalhar na pasta local `C:\Users\luanp\4i-simulador-previsibilidade`
  (identidade de commit: `4 Improvements <4improvementsportugal@gmail.com>`,
  passar com `git -c` porque não há user.email global).
- **Atenção:** por vezes aparecem commits "Update X.html" feitos diretamente
  no site do GitHub, historicamente sem alterações de conteúdo. Antes de fazer
  push: `git pull --rebase origin main`. Se o diff do remoto não for vazio,
  analisar antes de integrar.
- Existem cópias soltas em `C:\Users\luanp\*.html` (index/resultado/obrigado)
  que o Luan usa para testar: manter em sincronia com `cp` após cada alteração.
- O Git Credential Manager já está autenticado; `git push` funciona direto.

## Pendentes / backlog

1. **Embed do calendário GHL** no resultado.html (à espera da configuração
   Staff + Availability e do embed code).
2. Sugestão feita ao Luan (sem decisão): acrescentar opção "Menos de 80 mil
   euros" à pergunta de faturação, para não sujar dados de quem fatura menos.
3. Verificação real do telemóvel (OTP ou soft via GHL WhatsApp) ficou adiada.
4. Copys Meta: já foram entregues 6 ângulos de anúncio + 10 copys por vídeo +
   6 hooks agressivos para criativos de retenção (dores: comissão perdida,
   escravidão do telemóvel, inveja de pares, humilhação da imprevisibilidade,
   broker às cegas). Tom aprovado: cru e direto, sem travessões.

## Comandos úteis

```bash
# servir localmente
cd C:/Users/luanp/4i-simulador-previsibilidade && python -m http.server 8000
# (não há python no PATH desta máquina; abrir o index.html direto no browser funciona)

# verificar versão live vs local
curl -sS https://raw.githubusercontent.com/4improvementsportugal-ops/simulador-previsibilidade/main/index.html | grep -c "<marcador>"

# push seguro
git pull --rebase origin main && git push origin main
```
