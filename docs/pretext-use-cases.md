# Cas d'usage Pretext — compléments au mémo Codex

Ce document a deux parties :
- **Section A** : propositions absentes du mémo Codex
- **Section B** : implémentations concrètes pour les cas que Codex a identifiés
  mais laissés sans code

Architecture cible : normco.re — Lume + Deno, Markdown multi-langue
(fr / en / zh-hans / zh-hant), client React/Ant Design découplé.

---

## 1. Plugin Lume qui injecte les métriques textuelles dans le HTML statique

### Problème

Codex recommande d'utiliser pretext côté client React pour stabiliser les cartes.
Mais cela signifie que chaque visiteur exécute `prepare()` à chaque rendu initial —
même si le texte ne change jamais entre deux visites.

### Idée

Écrire un **plugin Lume** qui, lors du build, injecte les métriques pré-calculées
directement comme `data-*` dans le HTML généré. Le client lit ces attributs et
saute complètement `prepare()` + `layout()` au premier rendu.

```
prepare() ← exécuté une seule fois, au build (Deno + canvas headless)
layout()  ← exécuté côté client uniquement au resize
```

### Implémentation

```ts
// plugins/pretext_metrics.ts
import type { Site } from 'lume/core.ts'

/**
 * Plugin Lume qui mesure les titres et descriptions de chaque page
 * et injecte les résultats comme attributs data- dans les balises cibles.
 *
 * Nécessite : npm:@napi-rs/canvas (polyfill OffscreenCanvas pour Deno)
 */
export default function pretextMetrics(options: {
  selector: string    // ex: '[data-pretext]'
  font: string        // ex: '700 20px Inter'
  lineHeight: number  // ex: 28
  cardWidth: number   // ex: 280
}) {
  return (site: Site) => {
    site.process(['.html'], (pages) => {
      for (const page of pages) {
        const document = page.document
        if (!document) continue

        document.querySelectorAll(options.selector).forEach((el) => {
          const text = el.textContent?.trim() ?? ''
          if (!text) return

          // Les métriques sont calculées ici via un worker Deno
          // qui dispose du polyfill canvas (voir scripts/pretext-worker.ts)
          const metrics = globalThis.__pretextSync?.(text, options.font, options.cardWidth, options.lineHeight)
          if (!metrics) return

          el.setAttribute('data-line-count', String(metrics.lineCount))
          el.setAttribute('data-text-height', String(Math.ceil(metrics.height)))
        })
      }
    })
  }
}
```

Côté client React, le hook devient quasi-gratuit au montage initial :

```ts
// hooks/useTextLayout.ts
export function useTextLayout(el: HTMLElement | null) {
  // Lecture directe des attributs injectés au build — zéro calcul
  const lineCount = Number(el?.dataset.lineCount ?? 0)
  const height = Number(el?.dataset.textHeight ?? 0)

  // Au resize seulement, on recalcule via pretext
  const [live, setLive] = useState({ lineCount, height })
  useLayoutEffect(() => {
    if (!el) return
    const observer = new ResizeObserver(([entry]) => {
      const w = entry.contentRect.width
      const text = el.textContent ?? ''
      const font = el.dataset.font ?? '700 20px Inter'
      const lh = Number(el.dataset.lineHeight ?? 28)
      const prepared = prepare(text, font)
      setLive(layout(prepared, w, lh))
    })
    observer.observe(el)
    return () => observer.disconnect()
  }, [el])

  return live
}
```

### Valeur ajoutée vs mémo Codex

Codex propose pretext uniquement comme outil runtime. Ici, le coût de
`prepare()` est amorti sur l'ensemble des visiteurs en le déplaçant au build.
Sur un blog statique où les textes changent rarement, c'est le bon endroit.

---

## 2. Troncature des résumés par nombre de lignes, pas par nombre de caractères

### Problème

Lume et la plupart des pipelines Markdown tronquent les excerpts par nombre de
caractères (ex: 160 chars). En zh-hans, 160 caractères = 5–6 lignes visuelles
dans une carte à 280px. En français ou anglais, les mêmes 160 chars = 2–3 lignes.
Le résultat : des cartes visuellement incohérentes selon la langue.

### Idée

Un filtre Lume qui tronque intelligemment par `lineCount` cible plutôt que par
longueur de chaîne, en tenant compte de la fonte et de la largeur de la carte.

```ts
// plugins/excerpt_by_lines.ts
import { prepare, layout, layoutWithLines } from 'npm:@chenglou/pretext'

const FONT_BY_LANG: Record<string, string> = {
  fr:        '400 14px Inter',
  en:        '400 14px Inter',
  'zh-hans': '400 14px "Noto Sans SC"',
  'zh-hant': '400 14px "Noto Sans TC"',
}

/**
 * Retourne le texte tronqué pour tenir en exactement `maxLines` lignes
 * visuelles dans une carte de `cardWidth` pixels.
 */
export function excerptByLines(
  text: string,
  lang: string,
  cardWidth: number,
  maxLines: number,
  lineHeight = 22
): string {
  const font = FONT_BY_LANG[lang] ?? FONT_BY_LANG.en
  const prepared = prepare(text, font)
  const { lineCount } = layout(prepared, cardWidth, lineHeight)
  if (lineCount <= maxLines) return text

  // Coupe progressivement jusqu'à tenir
  const { lines } = layoutWithLines(prepared, cardWidth, lineHeight)
  const keepLines = lines.slice(0, maxLines)
  const lastLine = keepLines.at(-1)
  if (!lastLine) return text

  // Récupère le texte jusqu'à la fin de la dernière ligne retenue
  const cutIndex = lastLine.range[1]
  return text.slice(0, cutIndex).trimEnd() + '…'
}
```

Enregistrement comme filtre Lume :

```ts
// _config.ts
import { excerptByLines } from './plugins/excerpt_by_lines.ts'

site.filter('excerptByLines', (text: string, lang: string) =>
  excerptByLines(text, lang, 280, 3)
)
```

Usage dans un template Vento :

```vento
{{ post.description |> excerptByLines(lang) }}
```

### Valeur ajoutée vs mémo Codex

Codex ne mentionne pas du tout la troncature. C'est pourtant l'endroit où le
multilinguisme crée le plus d'incohérence visuelle concrète, et c'est un usage
**build-time pur** — aucun JS client requis.

---

## 3. Re-layout chirurgical après font swap

### Problème

Les navigateurs swappent les fontes web après le premier rendu (FOUT). Si pretext
mesure pendant la fonte de fallback, toutes les hauteurs calculées sont fausses
jusqu'au swap. Recalculer toutes les cartes après `fonts.ready` est coûteux et
provoque un flash visible.

### Idée

N'invalider que les éléments dont la mesure a réellement changé entre la fonte
de fallback et la fonte finale.

```ts
// src/scripts/font-swap-relayout.ts
import { prepare, layout } from '@chenglou/pretext'

const FALLBACK_FONT = '700 20px system-ui'
const FINAL_FONT    = '700 20px Inter'

async function relayoutAfterFontSwap() {
  // Mesures initiales avec la fonte de fallback
  const cards = Array.from(document.querySelectorAll<HTMLElement>('.post-card'))
  const snapshots = cards.map((card) => {
    const title = card.dataset.title ?? ''
    const w = card.offsetWidth - 32
    const { height } = layout(prepare(title, FALLBACK_FONT), w, 28)
    return { card, title, w, height }
  })

  // Attendre la fonte finale
  await document.fonts.ready

  // Re-mesurer et ne patcher que ce qui a changé
  let changed = 0
  for (const { card, title, w, height: oldHeight } of snapshots) {
    const { height: newHeight, lineCount } = layout(prepare(title, FINAL_FONT), w, 28)
    if (Math.abs(newHeight - oldHeight) < 1) continue // inchangé → skip

    card.style.minHeight = `${newHeight}px`
    card.dataset.lineCount = String(lineCount)
    changed++
  }

  if (changed > 0) {
    console.debug(`[pretext] re-layout après font swap : ${changed}/${cards.length} cartes`)
  }
}

document.fonts.ready.then(relayoutAfterFontSwap)
```

### Valeur ajoutée vs mémo Codex

Codex ne mentionne pas du tout le problème des fontes web et du FOUT. C'est
pourtant la source la plus fréquente d'inexactitude des mesures pretext en
production, en particulier pour les fontes CJK qui ont des fallbacks très
différents en largeur.

---

## 4. `measureNaturalWidth` pour les nuages de tags et taxonomies

### Problème

Les tags sur les pages archives et articles ont des largeurs calculées par le
navigateur via `width: fit-content`. Cette propriété CSS déclenche un reflow
par tag, ce qui est problématique quand on inspecte ou manipule leurs positions
programmatiquement (ex: un tag cloud ordonné par popularité avec positionnement
absolu).

### Idée

`measureNaturalWidth()` retourne la largeur intrinsèque d'un texte (la ligne la
plus large si le texte n'était pas contraint). C'est exactement `fit-content`
sans toucher le DOM.

```ts
// src/scripts/tag-cloud.ts
import { prepare, measureNaturalWidth } from '@chenglou/pretext'

interface Tag {
  label: string
  count: number
}

const TAG_FONT = '500 13px Inter'
const H_PADDING = 16 // padding gauche + droite du pill

function buildTagCloud(tags: Tag[], container: HTMLElement) {
  const containerWidth = container.offsetWidth

  // Mesure de toutes les largeurs en un seul batch — zéro reflow
  const measured = tags.map((tag) => {
    const prepared = prepare(tag.label, TAG_FONT)
    const pillWidth = measureNaturalWidth(prepared) + H_PADDING
    return { ...tag, pillWidth }
  })

  // Tri par largeur décroissante pour remplissage bin-packing simple
  measured.sort((a, b) => b.pillWidth - a.pillWidth)

  // Placement en lignes sans overflow
  let x = 0
  let y = 0
  const ROW_HEIGHT = 32

  for (const tag of measured) {
    if (x + tag.pillWidth > containerWidth) {
      x = 0
      y += ROW_HEIGHT + 8
    }
    const el = document.createElement('a')
    el.className = 'tag-pill'
    el.textContent = tag.label
    el.style.cssText = `
      position: absolute;
      left: ${x}px;
      top: ${y}px;
      width: ${tag.pillWidth}px;
    `
    container.appendChild(el)
    x += tag.pillWidth + 8
  }

  container.style.height = `${y + ROW_HEIGHT}px`
}
```

### Valeur ajoutée vs mémo Codex

Codex ne mentionne pas `measureNaturalWidth`. Ce cas d'usage est direct dans la
doc de pretext mais absent du mémo. Il est particulièrement pertinent pour les
pages `/tags` et les nuages de taxonomies où les largeurs varient fortement entre
les langues (tags CJK = très courts en px, tags français = longs).

---

## 5. OG images avec font sizing adaptatif au build

### Problème

Le plugin `og_images.ts` de Lume génère des images à taille fixe (ex: 1200×630).
Les titres longs débordent ou sont tronqués manuellement. En multilinguisme, un
titre zh-hans de 20 caractères prend 2× moins de place qu'un titre français de
même longueur sémantique.

### Idée

Un script Deno de build qui utilise pretext pour trouver le **font-size optimal**
pour chaque titre dans chaque langue, puis génère l'OG image avec la taille
exacte.

```ts
// scripts/og-fit-title.ts
import { prepare, layout } from 'npm:@chenglou/pretext'

const OG_WIDTH = 1200
const OG_CONTENT_WIDTH = 960  // marges = 120px de chaque côté
const OG_LINE_HEIGHT_RATIO = 1.25

interface FitResult {
  fontSize: number
  lineCount: number
  height: number
}

/**
 * Trouve le plus grand font-size pour lequel `title` tient en `maxLines`
 * lignes dans la zone utile de l'image OG.
 */
export function fitTitleToOG(
  title: string,
  fontFamily: string,
  maxLines = 2,
  sizeRange = { min: 36, max: 80 }
): FitResult {
  for (let size = sizeRange.max; size >= sizeRange.min; size -= 2) {
    const font = `700 ${size}px ${fontFamily}`
    const lineHeight = size * OG_LINE_HEIGHT_RATIO
    const prepared = prepare(title, font)
    const result = layout(prepared, OG_CONTENT_WIDTH, lineHeight)
    if (result.lineCount <= maxLines) {
      return { fontSize: size, ...result }
    }
  }
  // Fallback : taille minimale même si ça déborde
  const font = `700 ${sizeRange.min}px ${fontFamily}`
  const lh = sizeRange.min * OG_LINE_HEIGHT_RATIO
  return { fontSize: sizeRange.min, ...layout(prepare(title, font), OG_CONTENT_WIDTH, lh) }
}
```

Intégration dans le pipeline d'OG images de Lume :

```ts
// plugins/og_images_fitted.ts — wrapper du plugin og_images existant
import { fitTitleToOG } from '../scripts/og-fit-title.ts'

const FONT_FAMILY_BY_LANG: Record<string, string> = {
  fr: 'Inter',
  en: 'Inter',
  'zh-hans': 'Noto Sans SC',
  'zh-hant': 'Noto Sans TC',
}

site.process(['.html'], (pages) => {
  for (const page of pages) {
    const lang = String(page.data.lang ?? 'fr')
    const title = String(page.data.title ?? '')
    const fontFamily = FONT_FAMILY_BY_LANG[lang] ?? 'Inter'

    const fit = fitTitleToOG(title, fontFamily)

    // Injecte dans les données de page pour que le template OG les lise
    page.data.og_font_size = fit.fontSize
    page.data.og_title_height = Math.ceil(fit.height)
  }
})
```

### Valeur ajoutée vs mémo Codex

Codex cite les "layouts éditoriaux avancés" comme expérimental. Ce cas est
**concret, build-time, et résout un vrai problème** (débordement de titres dans
les OG images) avec la logique exacte pour laquelle pretext a été conçu — y
compris la sensibilité CJK.

---

## Synthèse des angles non couverts par Codex

| Cas | Moment | Bénéfice principal |
|-----|--------|--------------------|
| Plugin Lume + `data-*` | Build | Zéro coût `prepare()` côté client |
| Troncature par lignes | Build | Cohérence visuelle multilinguisme |
| Re-layout post font swap | Runtime | Exactitude après FOUT, ciblé CJK |
| `measureNaturalWidth` + tags | Runtime | Tag cloud sans reflow |
| OG images adaptatives | Build | Titres toujours ajustés, toutes langues |

Tous les cas build-time supposent un polyfill canvas pour Deno
(`npm:@napi-rs/canvas` ou un step puppeteer dédié).

---

## Section B — Implémentations concrètes des cas Codex

Codex a bien identifié les problèmes suivants, mais sans code ni précisions
d'architecture. Cette section comble les blancs.

---

### B.1 Cartes d'articles stables — hook React complet

Codex : « mesurer titre + description avant rendu final pour réduire les sauts
de layout ». Voici l'implémentation avec `ResizeObserver` et gestion du cycle
de vie React :

```ts
// src/hooks/useCardLayout.ts
import { useRef, useState, useLayoutEffect, useMemo } from 'react'
import { prepare, layout } from '@chenglou/pretext'

const FONT_MAP: Record<string, { title: string; desc: string }> = {
  fr:        { title: '700 20px Inter',         desc: '400 14px Inter' },
  en:        { title: '700 20px Inter',         desc: '400 14px Inter' },
  'zh-hans': { title: '700 20px "Noto Sans SC"', desc: '400 14px "Noto Sans SC"' },
  'zh-hant': { title: '700 20px "Noto Sans TC"', desc: '400 14px "Noto Sans TC"' },
}

const CARD_H_PADDING = 32
const TITLE_LINE_HEIGHT = 28
const DESC_LINE_HEIGHT = 22

export function useCardLayout(
  containerRef: React.RefObject<HTMLElement>,
  title: string,
  description: string,
  lang: string
) {
  const [width, setWidth] = useState(0)
  const fonts = FONT_MAP[lang] ?? FONT_MAP.fr

  // Observe la largeur du container — déclenche recalcul au resize
  useLayoutEffect(() => {
    const el = containerRef.current
    if (!el) return
    const ro = new ResizeObserver(([entry]) => {
      setWidth(entry.contentRect.width)
    })
    ro.observe(el)
    setWidth(el.offsetWidth)
    return () => ro.disconnect()
  }, [containerRef])

  // Prépare les textes (memoïsé — ne re-run que si texte ou fonte change)
  const preparedTitle = useMemo(() => prepare(title, fonts.title), [title, fonts.title])
  const preparedDesc  = useMemo(() => prepare(description, fonts.desc), [description, fonts.desc])

  // Layout pur (arithmétique) — re-run à chaque resize
  return useMemo(() => {
    if (width === 0) return null
    const w = width - CARD_H_PADDING
    const titleMetrics = layout(preparedTitle, w, TITLE_LINE_HEIGHT)
    const descMetrics  = layout(preparedDesc, w, DESC_LINE_HEIGHT)
    return {
      titleLines: titleMetrics.lineCount,
      descLines:  descMetrics.lineCount,
      totalHeight: titleMetrics.height + descMetrics.height + 48,
    }
  }, [width, preparedTitle, preparedDesc])
}
```

Usage dans le composant carte :

```tsx
export function ArticleCard({ title, description, lang }: CardProps) {
  const ref = useRef<HTMLDivElement>(null)
  const metrics = useCardLayout(ref, title, description, lang)

  return (
    <div ref={ref} style={{ minHeight: metrics?.totalHeight }}>
      <Card title={title}>
        {/* Ant Design Paragraph avec truncation exacte par lineCount */}
        <Typography.Paragraph
          ellipsis={{ rows: Math.min(metrics?.descLines ?? 3, 3) }}
        >
          {description}
        </Typography.Paragraph>
      </Card>
    </div>
  )
}
```

**Point clé absent du mémo Codex** : la séparation `prepare` (memoïsé sur le
texte) et `layout` (re-calculé sur la largeur) correspond exactement au modèle
mental de React — les deux phases ont des dépendances différentes.

---

### B.2 Virtualisation — hauteurs pré-calculées pour `react-virtual`

Codex : « calculer des hauteurs textuelles fiables en amont pour alimenter une
stratégie de virtualisation ». Voici le lien avec `@tanstack/react-virtual` :

```ts
// src/hooks/usePostListHeights.ts
import { useMemo } from 'react'
import { prepare, layout } from '@chenglou/pretext'

interface Post {
  title: string
  description: string
  lang: string
}

const CARD_WIDTH = 680  // largeur colonne principale en px
const CARD_BASE_HEIGHT = 80  // padding, date, tags

export function usePostListHeights(posts: Post[]): number[] {
  return useMemo(() => {
    return posts.map(({ title, description, lang }) => {
      const isZh = lang.startsWith('zh')
      const titleFont = isZh ? '700 20px "Noto Sans SC"' : '700 20px Inter'
      const descFont  = isZh ? '400 14px "Noto Sans SC"' : '400 14px Inter'
      const w = CARD_WIDTH - 32

      const { height: th } = layout(prepare(title, titleFont), w, 28)
      const { height: dh } = layout(
        prepare(description, descFont),
        w,
        22
      )
      return Math.ceil(th + dh + CARD_BASE_HEIGHT)
    })
  }, [posts])
}
```

Branchement sur `@tanstack/react-virtual` :

```tsx
import { useVirtualizer } from '@tanstack/react-virtual'
import { usePostListHeights } from '../hooks/usePostListHeights'

export function PostList({ posts }: { posts: Post[] }) {
  const parentRef = useRef<HTMLDivElement>(null)
  const heights = usePostListHeights(posts)

  const virtualizer = useVirtualizer({
    count: posts.length,
    getScrollElement: () => parentRef.current,
    estimateSize: (i) => heights[i],  // hauteur exacte, pas une estimation
    overscan: 3,
  })

  return (
    <div ref={parentRef} style={{ overflow: 'auto', height: '80vh' }}>
      <div style={{ height: virtualizer.getTotalSize() }}>
        {virtualizer.getVirtualItems().map((item) => (
          <div
            key={item.key}
            style={{ transform: `translateY(${item.start}px)` }}
          >
            <ArticleCard {...posts[item.index]} />
          </div>
        ))}
      </div>
    </div>
  )
}
```

**Point clé absent du mémo Codex** : `estimateSize` dans `react-virtual`
accepte une fonction par index — c'est exactement le bon endroit pour injecter
des hauteurs pretext. Sans ça, le virtualizer utilise une heuristique fixe qui
cause des sauts de scroll.

---

### B.3 QA CI multi-langue — script Deno sans puppeteer

Codex propose ce cas sans donner d'implémentation. Voici une version qui évite
puppeteer en utilisant `@napi-rs/canvas` comme polyfill Deno/Node :

```ts
// scripts/check-text-overflow.ts
// deno run -A --unstable-ffi scripts/check-text-overflow.ts

import { createCanvas } from 'npm:@napi-rs/canvas'
import { parse } from 'npm:yaml'

// Patch globalThis pour que pretext trouve OffscreenCanvas
// @ts-ignore
globalThis.OffscreenCanvas = class {
  constructor(w: number, h: number) {
    return createCanvas(w, h)
  }
}

// Import après le patch
const { prepare, layout } = await import('npm:@chenglou/pretext')

interface CheckRule {
  selector: 'title' | 'description'
  maxLines: number
  cardWidth: number
  lineHeight: number
  font: (lang: string) => string
}

const RULES: CheckRule[] = [
  {
    selector: 'title',
    maxLines: 2,
    cardWidth: 280,
    lineHeight: 28,
    font: (lang) => lang.startsWith('zh')
      ? '700 20px "Noto Sans SC"'
      : '700 20px Inter',
  },
  {
    selector: 'description',
    maxLines: 3,
    cardWidth: 280,
    lineHeight: 22,
    font: (lang) => lang.startsWith('zh')
      ? '400 14px "Noto Sans SC"'
      : '400 14px Inter',
  },
]

async function* walkMarkdown(dir: string): AsyncGenerator<string> {
  for await (const entry of Deno.readDir(dir)) {
    const path = `${dir}/${entry.name}`
    if (entry.isDirectory) yield* walkMarkdown(path)
    else if (entry.name.endsWith('.md')) yield path
  }
}

const failures: string[] = []
let checked = 0

for await (const file of walkMarkdown('./content')) {
  const raw = await Deno.readTextFile(file)
  const fmMatch = raw.match(/^---\n([\s\S]*?)\n---/)
  if (!fmMatch) continue

  const fm = parse(fmMatch[1]) as Record<string, string>
  const lang = fm.lang ?? 'fr'

  for (const rule of RULES) {
    const text = fm[rule.selector]
    if (!text) continue

    const font = rule.font(lang)
    const { lineCount } = layout(prepare(text, font), rule.cardWidth, rule.lineHeight)
    checked++

    if (lineCount > rule.maxLines) {
      failures.push(
        `[${lang}] ${file}\n  ${rule.selector}: "${text}"\n  → ${lineCount} lignes (max ${rule.maxLines} à ${rule.cardWidth}px)`
      )
    }
  }
}

if (failures.length > 0) {
  console.error(`\n❌ ${failures.length} débordement(s) détecté(s) :\n`)
  failures.forEach((f) => console.error(f + '\n'))
  Deno.exit(1)
}

console.log(`✓ ${checked} vérifications — aucun débordement`)
```

Ajout dans `deno.json` :

```json
{
  "tasks": {
    "check:overflow": "deno run -A scripts/check-text-overflow.ts"
  }
}
```

**Point clé absent du mémo Codex** : l'utilisation de `@napi-rs/canvas` comme
polyfill évite d'avoir à lancer un navigateur headless. Le script tourne en
CI standard (GitHub Actions ubuntu) sans installation supplémentaire hormis
les deps Deno/npm.

---

### Synthèse Section B

| Proposition Codex | Manque comblé |
|---|---|
| Cartes stables | Séparation `prepare`/`layout` alignée sur le modèle React |
| Virtualisation | Branchement direct sur `@tanstack/react-virtual` via `estimateSize` |
| QA CI | Script Deno autonome avec polyfill canvas, sans puppeteer |
| CLS / ancrage scroll | → voir cas A.3 (re-layout post font swap) |
| Layouts éditoriaux | → voir cas A.5 (OG images) et A.4 (tag cloud) |
