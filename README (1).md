# Atualização automática da fila de research

A fila de relatórios do app deixou de ser 100% manual. O fluxo agora é:

```
Genial Analisa (site)  →  scraper/scrape.mjs  →  reports.json  →  app.html
                          (GitHub Actions,                        (fetch no boot,
                           de hora em hora)                        com fallback)
```

## Como funciona

**1. `scraper/scrape.mjs`** lê `https://analisa.genialinvestimentos.com.br/acoes/`,
abre os N relatórios mais recentes e extrai de cada um:

| Campo | De onde vem |
|---|---|
| `ticker` | do box **Leitura Dinâmica** (`[MULT3]`), não do título — é por isso que a nota da "Axia" entra corretamente como **ALUP11** |
| `rec` | Leitura Dinâmica: Comprar / Manter / Vender / Em Revisão |
| `precoAtual`, `precoAlvo` | Leitura Dinâmica. `R$ 0,00` (placeholder de recomendação suspensa, ex.: BRAV3) vira `null` |
| `publicado` | data do card + hora exata da página ("Publicado em 13 de Julho às 11:09:56") |
| `setor` | breadcrumb (`Ações > Metais > CSN`) |
| `analista` | link `/analistas/<slug>` |
| `empresa`, `assunto` | `<h1>` da página |
| `corpo` | parágrafos do artigo (o suficiente para a IA resumir no app) |

**2. `reports.json`** é gravado na raiz do projeto, no mesmo schema do array
`REPORTS`.

**3. `app.html`** faz `fetch('reports.json')` no boot. Se der certo, troca a
fila, reordena, recalcula os "há Xh" e mostra *"✓ sincronizado com o Genial
Analisa às HH:MM"* embaixo do título da fila. Se der errado (arquivo ausente,
JSON inválido, app aberto direto do disco), ele **cai silenciosamente no array
embutido** — a demo nunca fica sem dados.

## Rodando na mão

```bash
node scraper/scrape.mjs              # gera ./reports.json
node scraper/scrape.mjs --limit 15   # mais relatórios
node scraper/scrape.mjs --dry        # imprime, não escreve
```

Precisa de Node 18+ (usa `fetch` nativo). Zero dependências.

## Automação

`.github/workflows/atualizar-research.yml` roda o scraper de hora em hora em
dias úteis e commita o `reports.json` só quando algo mudou. Também dá para
disparar na mão pelo botão **Run workflow**.

Para a demo se atualizar sozinha, ela precisa ser **servida por HTTP**
(GitHub Pages, Vercel, Netlify, ou `python3 -m http.server` local). Aberta com
duplo clique (`file://`), o navegador bloqueia o `fetch` e o app usa o
fallback — comportamento esperado, mas a fila fica congelada no que está
embutido.

## Campos que o scraper *deriva* (não existem no site)

Dois campos do app não têm equivalente publicado no Genial Analisa e são
inferidos por regra — vale revisar se o comportamento agradar:

- **`relevancia`** (ALTA/MÉDIA/BAIXA): ALTA quando é prévia, OPA/M&A,
  recomendação em revisão, ou o potencial até o preço-alvo é ≥ 20%. MÉDIA se o
  potencial for ≥ 8%. Caso contrário, BAIXA.
- **`recAnterior`**: o site não expõe a recomendação anterior, então é
  preenchida igual à atual — ou seja, **o app não vai destacar mudanças de
  recomendação** (o pill "MANTER → COMPRAR") nos relatórios vindos do scraper.
  Para ter isso, o scraper precisaria guardar um histórico entre execuções.
  É um incremento fácil se você quiser.

## Fragilidade conhecida

O scraper depende do HTML do Genial Analisa. Se a Genial mudar o layout, o
parser pode parar de achar os campos. Por isso ele foi feito para **falhar de
forma segura**: se não extrair nenhum relatório, ele aborta com erro e **não
sobrescreve** o `reports.json` bom que já existe, e o app continua servindo a
última versão válida. Numa integração de verdade (webhook interno da Genial),
essa fragilidade desaparece.
