# Bunny Chaos! Loot Arena — Design Document (estado atual)

> Snapshot do que está **implementado hoje** no `index.html`. Os docs separados em `~/Desktop/` (`design.md` original do Forge Master, `design-level-scaling.md`, `design-forge-upgrade.md`) são as **specs** que pautaram a implementação. Este doc é a fotografia do código real.

---

## 1. Visão Geral

**Bunny Chaos! Loot Arena** é um arena survivor top-down com camada idle/RPG. O jogador controla um coelho que ataca automaticamente, mata inimigos em waves, acumula moeda e força progressivamente seus itens via um sistema de **forja com dados 5d6** (estilo Yahtzee) + um sistema de **upgrade com cooldown**.

- **Stack**: HTML/CSS/JS puro num único `index.html`. Sem framework, sem build step. Canvas 2D pra gameplay, HTML/CSS pros overlays/popups.
- **Plataforma alvo**: smartphones modernos (9:20 portrait), PWA-capable (add to home screen). Responsivo via `100dvh` + `env(safe-area-inset-*)`.
- **Idioma**: PT-BR.
- **Idle/Incremental**: o jogo gera recursos enquanto fechado via popup ZZZ. Combate ativa por padrão (AUTO ON).

---

## 2. Loop Principal de Jogo

```
[matar inimigos] → cenouras + XP (drops com delay 1s + magnetic pull)
        ↓
[acumular cenouras] → upgrade de items (cap Lv 100) + custo da forja
        ↓
[acumular alface offline (via ZZZ, 1/min)] → forjar com dados
        ↓
[rolar 5 dados] → combo → gera item (era + nível inicial + substats)
        ↓
[equipar item] → mais poder de combate → matar inimigos mais fortes
        ↓ (boss kill)
[wave clear / stage clear] → +cenouras bônus → próxima stage
```

---

## 3. Recursos & Moedas

| Recurso | Símbolo | Função | Fontes |
|---|---|---|---|
| **Alface** | 🥬 | Fuel da forja (1 alface = 1 rolagem inicial). Reroll grátis. | Idle offline (1/min via ZZZ). **Não dropa de inimigos.** |
| **Cenoura** | 🥕 | Moeda de upgrade (item level + forge level). | Drop de inimigos (1/normal, 20/boss) · stage clear bonus · idle offline (1/s via ZZZ) |
| **XP** | — | Level do hero (cura + xpToNext exponencial) | Drop XP gem por inimigo (e.xp escala com stage) |
| **Nível do hero** | LV | Cura ao subir + métrica visual | Sobe automaticamente quando XP enche |

**Estado inicial**: 100🥬, 0🥕.

### Drops & Pickup

- XP gems + 🥕 caem no chão na posição do inimigo morto.
- Ficam **1 segundo parados** (60 frames @ 60fps).
- Depois voam magneticamente direto pro player com velocidade acelerando de 2 → 8 unidades/frame. Independente de `pickupRange`.
- **Items** (drops de gear) — atualmente removidos (todo gear vem só da forja).

---

## 4. Sistema de Forja com Dados (5d6 Yahtzee)

### 4.1 Mecânica

- 5 dados de 6 faces, rolados de uma vez.
- Custo base: **1🥬** por rolagem.
- **1 reroll** disponível (grátis), com hold/travar individual em cada dado.
- Combo detectado por precedência tipo Yahtzee.

### 4.2 Combos & multiplicadores

| Combo | Mult | TierUp % | DoubleUp % | Prob exata |
|---|---|---|---|---|
| Cinco Iguais | ×10.5 | 100% | 25% | 0.077% |
| Quatro Iguais | ×5.5 | 50% | 0% | 1.93% |
| Full House | ×4.0 | 25% | 0% | 3.86% |
| Sequência (4 consec.) | ×3.5 | 15% | 0% | 15.43% |
| Três Iguais | ×3.0 | 10% | 0% | 15.43% |
| Dois Pares | ×2.5 | 5% | 0% | 23.15% |
| Um Par | ×2.0 | 2% | 0% | 37.04% |
| Lançamento Máx. (todos ≠) | ×1.5 | 0% | 0% | 3.09% |

Multiplicador médio esperado por forja: **×2.64**.

### 4.3 Efeitos do combo no item gerado

Três dimensões independentes:

**1. Tier-Up da era**: depois de sortear a era base via gaussiana do `forgeLevel`, o combo pode subir 1 nível de era (ou 2 com Cinco Iguais).

**2. Nível inicial do item (Lv 1-100)**:
```
percentil = (mult - 1.5) / 9
baseStart = round(percentil × 75 + 5)
variance = round(percentil × 8 + 2)
initialLevel = clamp(baseStart ± variance, 1, 100)
```

| Combo | Nível inicial médio |
|---|---|
| Lançamento Máx. | ~Lv 5 |
| Um Par | ~Lv 9 |
| Dois Pares | ~Lv 13 |
| Três Iguais | ~Lv 18 |
| Sequência | ~Lv 22 |
| Full House | ~Lv 26 |
| Quatro Iguais | ~Lv 38 |
| Cinco Iguais | ~Lv 80 |

**3. Roll Quality dos substats (0..1, fixo na forja)**:
```
percentil = (mult - 1.5) / 9
rollLow  = percentil × 0.6
rollHigh = 0.4 + percentil × 0.6
rollQuality = uniform(rollLow, rollHigh)
```

O valor exibido escala com o level do item:
```
substatValue = max × rollQuality × (0.4 + 0.6 × levelFactor(level))
```

### 4.4 UI inline da forja

Substitui o popup tradicional. Strip permanente no rodapé do play grid:

```
[D1] [D2] [D3] [D4] [D5]    [LV X]
[      ROLAR (1🥬)     ]    [AUTO]
       Combo · ×N            [🥬 N]
```

- Dados pintados na cor do combo (amarelo Par → vermelho Cinco)
- Botão muda contextualmente: ROLAR → REROLAR (após primeira rolagem) → FORJAR
- Após FORJAR, abre popup de comparação com substats vs item equipado

---

## 5. Sistema de Itens

### 5.1 Slots de equipamento (8)

| Slot | Stat primário | baseStat0 (Era 1 Lv 1) |
|---|---|---|
| weapon | dmg | 3.0 |
| helmet | maxHp | 12 |
| armor | maxHp | 18 |
| gloves | atkSpeed | 0.05 |
| belt | maxHp | 8 |
| boots | moveSpeed | 0.08 |
| amulet | crit | 0.05 |
| ring | dmg | 2.0 |

### 5.2 Eras (10 tiers)

Cada item nasce em uma era, determinada pela gaussiana do `forgeLevel` + tier-up do combo.

| Idx | Nome | Unlock (forgeLvl) | Substat slots | Cor |
|---|---|---|---|---|
| 0 | Primitive | 1 | 0 | #888780 |
| 1 | Medieval | 4 | 0 | #5F5E5A |
| 2 | Early-Modern | 7 | 0 | #BA7517 |
| 3 | Modern | 11 | 1 | #378ADD |
| 4 | Space | 15 | 1 | #185FA5 |
| 5 | Interstellar | 19 | 1 | #0C447C |
| 6 | Multiverse | 23 | 2 | #7F77DD |
| 7 | Quantum | 27 | 2 | #534AB7 |
| 8 | Underworld | 30 | 2 | #D85A30 |
| 9 | Divine | 33 | 2 | #1D9E75 |

### 5.3 Gaussiana de era

```
peak(era) = era.unlock + ERA_PEAK_OFFSET (2)
peso(era, forgeLvl) = max(0.5, 100 × exp(-(forgeLvl - peak)² / 8))
                       se forgeLvl ≥ era.unlock, senão 0
```

ERA_PEAK_OFFSET ajustado de 4 → 2 pra Divine dominar no lvl 35 (em 4, Underworld dominava).

### 5.4 Stat scaling (level dentro da era)

```
levelFactor(L) = ((L-1)/99)^1.4         // 0 em L=1, 1 em L=100, back-loaded
baseStat(slot, era, level) =
   SLOTS[slot].baseStat0
 × ERA_MULT^(era-1)
 × (1 + levelFactor × WITHIN_GROWTH)

ERA_MULT = 1.43
WITHIN_GROWTH = 0.40
```

Calibrado pra preservar a constraint **Era N Lv.100 < Era N+1 Lv.1** (margem ~2.1% em cada transição). Total Era 1 Lv 1 → Era 10 Lv 100 = ~28×.

### 5.5 Substats (13 tipos)

Cada item, dependendo da era, recebe 0, 1 ou 2 substats sorteados sem repetição.

| ID | Nome | Max | Mapeamento → stat de combate |
|---|---|---|---|
| CRIT_CHANCE | Crit Chance | 12% | crit (× v/100) |
| CRIT_DAMAGE | Crit Damage | 100% | dmg (× v/100) |
| ATTACK_SPEED | Attack Speed | 40% | atkSpeed (× v/100) |
| DOUBLE_CHANCE | Double Chance | 40% | projectiles (× v/40) |
| DAMAGE | Damage | 15% | dmg (× v/10) |
| SKILL_DAMAGE | Skill Damage | 30% | dmg (× v/50) |
| RANGED_DAMAGE | Ranged Damage | 15% | dmg (× v/30) |
| MELEE_DAMAGE | Melee Damage | 50% | dmg (× v/100) |
| BLOCK | Block | 5% | dodge (× v/200) |
| LIFESTEAL | Lifesteal | 20% | pickupRange (× v) |
| HEALTH_REGEN | Health Regen | 6% | xpMult (× v/100) |
| SKILL_COOLDOWN | Skill Cooldown | 7% | atkSpeed (× v/200) |
| HEALTH | Health | 15% | maxHp (× v) |

### 5.6 Upgrade de item (Lv 1 → 100)

```
upgradeCost(era, currentLevel) =
   BASE_ESSENCE (10)
 × ESSENCE_ERA_MULT (1.6) ^ era
 × (1 + (currentLevel/100)² × COST_LEVEL_FACTOR (9))
```

Custo total Era 1 → Lv 100: ~3.900 cenouras.
Custo total Era 10 → Lv 100: ~268.500 cenouras.

Botões no popup do item: `MELHORAR` (1 nível), `×10` (10 níveis agregado).

---

## 6. Combate

### 6.1 Tipos de arma

Arma equipada determina se o ataque é melee ou ranged:
- **Melee**: swing de espada-cenoura em arco frontal. Hit area cresce com `projectiles`.
- **Ranged**: dispara N projéteis (N = `projectiles`).

### 6.2 Stats agregados

`arenaTotalStat(name)` soma:
1. Stat primário do item se `it.stat === name` (escalado por era × level via `baseStat`)
2. Cada substat convertido via `SUBSTAT_TO_ARENA`

Stats disponíveis: `dmg`, `maxHp`, `atkSpeed`, `crit`, `dodge`, `moveSpeed`, `pickupRange`, `projectiles`, `xpMult`.

### 6.3 IA do AUTO

Player AUTO ativo por padrão. Decisão por frame:
- Identifica inimigo mais próximo
- Se muito perto (< idealNear): foge
- Se muito longe (> idealFar): aproxima
- Se na zona ideal (entre near e far): kiting perpendicular
- Bias suave pro centro do grid (anti-cantonado)
- Anti-stuck: detecta se não andou em 48 frames e força escape pro centro

Botão AUTO no canto inferior direito do grid alterna ON/OFF (cor verde quando ativo).

### 6.4 Hurt / Dodge / Respawn

- Contato com inimigo: aplica dano se `dodge` falha
- Dodge fraction = `arenaTotalStat('dodge')`. Roll < dodge = esquiva
- Após dano: 40 frames de invuln
- HP zera → respawn no centro, HP cheio, volta pra wave 1 da stage atual
- Se morreu lutando boss: marca `failedBossCurrentStage` → botão "DESAFIAR BOSS" aparece nas waves seguintes

### 6.5 Inimigos

Pool escala com stage:
- Stage 1: 2× slime
- Stage 2+: slime + bat
- Stage 5+: + skeleton
- Stage 8+: + knight
- Stage 12+: knight enriquecido
- Stage 16+: knight dominante

HP/dmg escalam: `1.15^(stage-1)` para inimigo, ×1.2 extra pra boss.

---

## 7. Wave / Stage System

- **5 waves por stage** (4 normais + 1 boss).
- Wave 1-3: spawn de N inimigos da pool da stage.
- Wave 4: limpa → avança automaticamente pro boss (1ª vez). Se já perdeu pro boss da stage, espera o player clicar "DESAFIAR BOSS".
- Wave 5 (boss): inimigo único spawnado no topo.
- Boss derrotado → "STAGE CLEAR!" banner por 2s → próxima stage.

Stage clear bonus: `(5 + stage) × 5` cenouras 🥕.

Label de stage: formato `{chapter}-{within}` onde chapter = ceil(stage/5).

---

## 8. Upgrade da Forja (forgeLevel 1 → 35)

### 8.1 Custo + Cooldown

Substitui o sistema antigo de unlock por kills. Cada nível custa cenouras + tem timer:

```
FORGE_UPGRADES[34 entries] com cost (cenouras) + timeSec (segundos)
```

Curva base (sem multiplicadores):
- Lvl 1→2: 10🥕 / 5s
- Lvl 20→21: 17.000🥕 / 8m30s
- Lvl 34→35: 350.000🥕 / 20m

Total base: ~2M cenouras, ~4h.

### 8.2 Multiplicadores globais

```js
const FORGE_COST_MULT = 20.0;   // atual
const FORGE_TIME_MULT = 20.0;   // atual
```

Ajuste em UM lugar → toda a curva muda. Aplicados via `getForgeUpgradeInfo(level)` que retorna `{cost, timeSec}` já multiplicado.

Modos sugeridos:
- EXPRESS (0.1×): ~200k 🥕, ~26 min
- STANDARD (1×): ~2M 🥕, ~4h
- **Atual (20×)**: ~40M 🥕, ~80h
- LONG_BURN (5× / 10×): ~10M 🥕, ~44h

### 8.3 Fluxo

1. Player clica botão LV no widget de forja → popup mostra Custo + Tempo do próximo upgrade.
2. Click "Iniciar (N🥕)" → debita cenouras, marca `arena.forgeUpgradeStartedAt = Date.now()`, persistido em localStorage.
3. Popup mostra progress bar + countdown atualizando a cada 500ms.
4. Timer baseado em `Date.now()` absoluto — sobrevive a reload e fechamento da aba.
5. Quando expira: botão vira "Coletar upgrade" (ou auto-aplica via tick global 1Hz se popup fechado).
6. Click Coletar → `arena.forgeLevel++`, persiste no localStorage.

### 8.4 Persistência

- `bunnyChaos.forgeLevel` — nível atual
- `bunnyChaos.forgeUpgradeProgress` — `{startedAt, target, timeSec}` se em progresso
- Boot restaura ambos, auto-aplica se completou offline.

---

## 9. Offline Rewards (popup ZZZ)

### 9.1 Acúmulo

Recursos passivos enquanto o jogo está **fechado**:
- 🥕 cenoura: **1 por segundo** (cap 12h)
- 🥬 alface: **1 por minuto**

### 9.2 Pending pool persistido

Para garantir que o reward **nunca se perde** mesmo se o player fechar o popup sem coletar:

```
localStorage:
  bunnyChaos.lastSeen      → timestamp do último snapshot
  bunnyChaos.offlinePending → {carrots, lettuce} acumulado a coletar
  bunnyChaos.offlinePendingSec → segundos totais não coletados
```

`accruePendingFromLastSeen()` drena o delta desde `lastSeen` → pending pool → atualiza `lastSeen` pra agora. Chamado em: boot, intervalo 5s, `visibilitychange`, `pagehide`, `beforeunload`, abertura do popup.

`collectOfflineRewards()` move pending → `arena.carrots`/`arena.lettuce`, zera pending.

### 9.3 UI

Botão **ZZZ** no canto inferior esquerdo do grid (simétrico ao AUTO). Glow azul pulsante quando há rewards pendentes.

Popup estilo Forge Master com:
- Header roxo: título + tempo acumulado (verde)
- Dois ícones circulares com taxas: 🥕 1/s + 🥬 1/m (rings conic-gradient ciclando como relógios — 1/s e 1/min)
- Body claro: valores totais + botão "Coletar" (azul)
- X vermelho redondo pra fechar sem coletar (pending persiste)

---

## 10. UI / HUD

### 10.1 Layout 50/50

```
┌─────────────────────────────────┐
│ [LV+XP]   [FASE X-Y]   [🥕 N]   │ ← HUD pills flutuantes
│ [⚔️ N]                          │
│                                 │
│        ┌──────────────┐         │
│        │   ♥ HP bar   │         │
│        │   🐰 (auto)  │         │ ← Play grid (50% do frame)
│        │              │         │
│        │              │         │
│        │              │         │
│        ZZZ [💀 N]    AUTO       │
├─────────────────────────────────┤
│ [Lv1] [helmet] [armor] [gloves] │ ← Items strip (8 slots)
│       │etc        ...    [ring] │
│                                 │
│       [D] [D] [D] [D] [D] [LV]  │ ← Forge dice widget
│       [   ROLAR (1🥬)    ][AUTO]│
│             Combo · ×N    [🥬 N]│
│                                 │
│ [LOJA HERÓI EVOLUÇÃO COMBATE …] │ ← Tab bar (7 abas)
└─────────────────────────────────┘
```

**Frame**: respeita `safe-area-inset-top/bottom` do iOS. `width = 100vw - inset-x`, `height = 100dvh - inset-y`. Sem letterbox no mobile (acompanha aspect ratio do device).

**ARENA.BOT_Y = H/2** (fixo): play grid ocupa exatamente a metade superior; UI ocupa a metade inferior em qualquer tamanho de smartphone.

### 10.2 HUD pills (top)

**Player column (esquerda, empilhada)**:
- LV + XP: badge circular com level + barra de XP cur/next (largura 80, altura 31)
- ⚔️ Power: valor agregado do `arenaPower()` com pulse no equip

**STAGE pill (centro)**: `FASE X-Y` + 5 wave dots embaixo

**🥕 Cenouras pill (direita)**: counter dourado

### 10.3 Bottom-of-grid

- **💀 Kills**: pill centro do bottom do grid
- **AUTO**: canto inferior direito, ativo verde
- **ZZZ**: canto inferior esquerdo, glow azul se tem reward

### 10.4 Popup de item

Click num slot → popup centralizado verticalmente na metade dim de baixo:
- Header: ícone + nome (cor da era) + meta (era · combo)
- Slot label (canto direito)
- Stat primário (escalado por level)
- Barra de level X/100
- Badge "TIER UP" se houve
- Lista de substats com bars de progresso
- Botões MELHORAR / ×10 (custo em cenouras)

Capture-phase click gate: enquanto popup aberta, qualquer click fora do card só fecha (não dispara o handler embaixo).

### 10.5 Items strip (8 slots no canvas)

- Slot vazio: ícone monocromático chapado (cinza)
- Slot equipado: background pintado **na cor da era** com overlay escuro (legibilidade) + ícone colorido + "Lv X" em ciano

---

## 11. Constantes Ajustáveis (no `index.html`)

Tudo balanceável em um lugar:

| Const | Valor | Função |
|---|---|---|
| `FORGE_BASE_COST` | 1 | 🥬 por rolagem inicial |
| `FORGE_REROLL_COST` | 0 | 🥬 por reroll |
| `FORGE_MAX_REROLLS` | 1 | Cap de rerolls |
| `FORGE_COST_MULT` | 20.0 | Multiplicador global de custo (forge upgrade) |
| `FORGE_TIME_MULT` | 20.0 | Multiplicador global de tempo (forge upgrade) |
| `MAX_ITEM_LEVEL` | 100 | Cap de level por item |
| `ERA_MULT` | 1.43 | Multiplicador entre eras (no `baseStat`) |
| `WITHIN_GROWTH` | 0.40 | Crescimento Lv1→Lv100 dentro da era |
| `LEVEL_CURVE_EXP` | 1.4 | Expoente da curva de levelFactor |
| `BASE_ESSENCE` | 10 | Custo base Lv1→2 (item upgrade) |
| `ESSENCE_ERA_MULT` | 1.6 | Custo cresce por era |
| `COST_LEVEL_FACTOR` | 9 | Quão mais caro fica o último upgrade |
| `OFFLINE_CARROT_PER_SEC` | 1 | Taxa offline cenoura |
| `OFFLINE_LETTUCE_PER_SEC` | 1/60 | Taxa offline alface |
| `OFFLINE_CAP_MS` | 12h | Cap de acúmulo offline |
| `ERA_PEAK_OFFSET` | 2 | Pico da gaussiana de era = unlock + offset |
| `WAVES_PER_STAGE` | 5 | 4 normais + 1 boss |

---

## 12. Persistência (localStorage)

| Chave | Valor |
|---|---|
| `bunnyChaos.lastSeen` | timestamp do último snapshot online |
| `bunnyChaos.offlinePending` | `{carrots, lettuce}` acumulado |
| `bunnyChaos.offlinePendingSec` | segundos não coletados |
| `bunnyChaos.forgeLevel` | nível atual da forja |
| `bunnyChaos.forgeUpgradeProgress` | `{startedAt, target, timeSec}` se em progresso |

Tudo o mais (carrots, lettuce, items, stage, killCount, ...) **reseta a cada sessão** (não persistido). Em iterações futuras pode mover pra save state completo.

---

## 13. Estados Removidos (legacy)

Para histórico e contexto:

- **Tower Ascension** e **Carrot Cave** — minigames anteriores, removidos. Boot direto na Loot Arena.
- **Dice combat skills** (PAIR=spin, FULL=orbiter, FIVE=nuke...) — sistema antigo onde dice rolavam a cada 5s e disparavam habilidade. Removido junto com o HUD de dados no topo.
- **Item drops de inimigos** — bosses davam 100%, normais 6%. Removido — items só via forja.
- **Lettuce drop de inimigos** — bosses davam 10, normais 1. Removido — alface só vem do idle offline.
- **Tier unlock por kills** — antiga `FORGE_THRESHOLDS[]` baseada em kill count. Substituída por cenouras + cooldown.
- **`triggerArenaAbility`** function — toda a lógica de combos de combate.

---

## 14. Arquivos Externos de Referência

- `~/Desktop/design.md` — spec original do Forge Master (Yahtzee + 10 eras + substats)
- `~/Desktop/design-level-scaling.md` — spec do level scaling (Lv 1-100 nos items, baseStat formula, upgrade cost)
- `~/Desktop/design-forge-upgrade.md` — spec do upgrade da forja (cost + cooldown, calibração standard / express / long-burn)
- `~/Desktop/design-forge-upgrade-pt-cenouras-com-progressao-acelerada-em-3-fases-prototype-do-codex.md` — variante PT-BR do anterior

---

## 15. Repositório

- **GitHub**: https://github.com/fernandopepe-wls/bunny-chaos
- **Branch principal**: `main`
- **Deploy**: GitHub Pages (atualiza automaticamente em cada push)

---

*Documento gerado por consolidação do estado atual do `index.html`. Atualizar conforme novas features forem implementadas.*
