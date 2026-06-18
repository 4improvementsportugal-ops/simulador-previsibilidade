# Simulador de Previsibilidade Imobiliária — 4 Improvements

Funil de 5 perguntas + diagnóstico por cluster de maturidade comercial, desenhado para captar consultores, team leaders, diretores comerciais e brokers do mercado imobiliário português.

## Estrutura

- **`index.html`** — Hero + funil (5 perguntas + 3 passos de contacto). No submit: guarda em `sessionStorage`, envia webhook, dispara `Lead` no pixel, redireciona para `resultado.html`.
- **`resultado.html`** — Lê `sessionStorage`, renderiza diagnóstico personalizado por cluster + perfil + foco. VSL gating: vídeo do Pedro Ferreira (Google Drive) destranca diagnóstico + calendário ao terminar.

## Deploy (GitHub Pages)

```bash
# 1) Push para o repo
git remote add origin https://github.com/4improvementsportugal-ops/<nome-do-repo>.git
git branch -M main
git push -u origin main

# 2) GitHub → Settings → Pages → Source: deploy from branch `main` / root
# 3) URL ficará: https://4improvementsportugal-ops.github.io/<nome-do-repo>/
```

## Placeholders a substituir antes de ir para produção

| Placeholder | Ficheiro | Onde |
|---|---|---|
| `[META_PIXEL_ID]` | `index.html` | 2 ocorrências no `<head>` (script + noscript) |
| `[WEBHOOK URL]` | `index.html` + `resultado.html` | `const WEBHOOK_URL` no topo do `<script>` |

## Configurações importantes

- **Vídeo VSL** — file ID do Google Drive: `17tmkYdQE89f001Pv1bMFJaBf5AiwYNgT` (já configurado). O ficheiro tem de estar partilhado como *"Qualquer pessoa com o link pode visualizar"*.
- **Duração do gating** — `VSL_DURATION_SECONDS = 90` em `resultado.html`. Ajustar à duração real do vídeo do Pedro Ferreira.
- **WhatsApp/Telefone** — `+351 963 934 488` (já configurado nos dois ficheiros).

## Sistema de scoring

3 perguntas pontuadas (estado da operação, velocidade de resposta, IA & BD), max 12 pontos. 4 clusters de maturidade:

| Score | Cluster | Posicionamento de venda |
|---|---|---|
| 0–3 | Operação Reativa | Está a perder negócios por resposta lenta + sem follow-up |
| 4–6 | Potencial Mal Aproveitado | Tem leads e BD mas falta sistema para os trabalhar |
| 7–9 | Operação Sem Previsibilidade | Tem estrutura mas vive no improviso, sem indicadores |
| 10–12 | Operação Pronta para Escalar | Tem peças isoladas, falta integração para escalar |

## Payload do webhook (lead)

```json
{
  "timestamp": "2025-...",
  "origem": "4 Improvements — Simulador Previsibilidade Imobiliária",
  "nome": "...",
  "empresa": "...",
  "email": "...",
  "telemovel": "+351...",
  "perfil": "Consultor | Team Leader | Diretor Comercial | Broker / Agência",
  "perfil_slug": "consultor | team_leader | diretor | broker",
  "estado_operacao": "...",
  "velocidade_resposta": "...",
  "ia_base_dados": "...",
  "foco_90d": "...",
  "foco_90d_slug": "velocidade | followup | recuperar_bd | angariacoes | previsibilidade | escalar",
  "score_maturidade": 0,
  "score_max": 12,
  "cluster_slug": "reativa | potencial | semprev | pronta",
  "cluster_label": "...",
  "raw": { "...todos os campos brutos..." }
}
```

## Stack

- HTML + CSS + JS vanilla (sem dependências)
- Google Fonts (Space Grotesk + Inter + JetBrains Mono)
- Google Drive embed para o VSL
- Mobile-first, responsive até 1024px+
