# Projeto FPS Competitivo (codinome a definir)

> FPS tático 5v5 free-to-play que combina a **fidelidade visual e o netcode do CS2**, o **gunplay e o ritmo do CrossFire**, e a **curadoria de arsenal do Valorant**. Ambientação 100% brasileira — nicho inexplorado nos FPS competitivos.

**Status:** documento de fundação / pré-produção. Nome do jogo ainda a definir.

---

## Sumário

- [Pilares de design](#pilares-de-design)
- [Regras de ouro](#regras-de-ouro-inegociáveis)
- [Stack técnica](#stack-técnica)
- [Arquitetura](#arquitetura)
- [Estrutura de repositórios](#estrutura-de-repositórios)
- [Modos de jogo](#modos-de-jogo)
- [Mapas](#mapas)
- [Arsenal](#arsenal-27-armas)
- [Sistema de recoil](#sistema-de-recoil)
- [Modelo de dano e penetração](#modelo-de-dano-e-penetração)
- [Granadas](#granadas)
- [Sistema de compra](#sistema-de-compra)
- [Economia da ranqueada](#economia-da-ranqueada)
- [Bíblia de mapas (level design)](#bíblia-de-mapas)
- [Custos de hospedagem](#custos-de-hospedagem)
- [Roadmap](#roadmap)
- [Decisões em aberto](#decisões-em-aberto)

---

## Pilares de design

1. **Visual CS2:** PBR, texturas fotorrealistas (Quixel Megascans), iluminação Lumen.
2. **Gunplay CrossFire:** recoil vertical-dominante e controlável, movimento ágil, troca de arma rápida, TTK baixo.
3. **Curadoria Valorant:** arsenal enxuto, cada arma com papel claro, nomes fictícios na fase comercial.
4. **Netcode CS2:** servidor autoritativo, 64 tick, lag compensation, validação server-side de tudo.

## Regras de ouro (inegociáveis)

- **Zero vantagem comprável.** Variantes lendárias de armas (tradição CF) são só visuais — stats idênticos sempre, inclusive no corpo a corpo (machado = skin da faca).
- **Sem armas travadas por lado.** TR e CT compram do mesmo arsenal (modelo CrossFire/Valorant). Pares equivalentes têm preço igual e viram escolha de estilo: AK-47 vs M4A1 ($2.900), Galil vs FAMAS ($1.900), SG 553 vs AUG ($3.100).
- **Cliente nunca decide hit.** Servidor valida dano, spread e penetração.
- **Nomes de armas:** desenvolver com nomes reais; renomear levemente no lançamento (marcas como H&K, Barrett, Glock e Desert Eagle são registradas e a H&K processa estúdios).

## Stack técnica

| Camada | Tecnologia |
|---|---|
| Jogo (cliente + servidor) | Unreal Engine 5 + C++ (cliente Windows, dedicated server Linux) |
| Backend de plataforma | .NET 8 / C#, PostgreSQL, Redis, Docker |
| Site / portal | Next.js (perfil, leaderboard, loja) |
| Serviços externos gratuitos | Epic Online Services (matchmaking, lobbies, voz, Easy Anti-Cheat) |
| Orquestração de partidas | Agones (Kubernetes) ou GameLift no lançamento; VPS próprio no protótipo |

> **Licença UE5:** gratuita, 5% de royalty só sobre receita bruta vitalícia acima de US$ 1 milhão.

## Arquitetura

```
Cliente UE5 (C++, Windows, anti-cheat)
        |
        v
Servidor dedicado UE5 (autoritativo, Linux, 64 tick)
        |
        v
Backend de plataforma (.NET 8, Docker)
  ├── Auth e contas (JWT + Steam ID)
  ├── Matchmaking (filas Redis + MMR)
  └── Inventário e stats (skins, histórico)
        |
        v
PostgreSQL (contas, partidas)  +  Redis (filas, sessões)
```

**Fluxo de uma partida ranqueada:** fila ranqueada (Redis, busca por MMR) → lobby 5v5 balanceado + veto de mapa → servidor alocado por região → partida MR12 (vence quem fizer 13) → pós-partida (ELO, stats, replay).

## Estrutura de repositórios

```
fps-game/                          # Repositório UE5 (Perforce ou Git LFS)
├── Source/
│   ├── FPSGame/                   # Módulo principal (C++)
│   │   ├── Weapons/               # WeaponBase, RecoilComponent, BallisticsSubsystem
│   │   ├── Characters/            # Movimento estilo CrossFire (strafe, swap rápido)
│   │   ├── GameModes/             # SearchDestroyMode, TeamDeathmatchMode, FreeForAllMode
│   │   ├── Rounds/                # Economia, timer, plant/defuse, MR12
│   │   └── Net/                   # Lag compensation, validação server-side
│   └── FPSGameServer/             # Target do dedicated server (Linux)
├── Content/
│   ├── Maps/                      # SD_*, TDM_*, CIS_*
│   ├── Weapons/                   # Meshes/texturas PBR (Megascans + Substance)
│   ├── Characters/
│   └── UI/                        # HUD, menus, placar
└── Config/Weapons/                # DT_Weapons.csv — balanceamento (editar planilha, não recompilar)

fps-platform/                      # Monorepo backend (.NET + Next.js)
├── src/
│   ├── Auth.Api/                  # login, JWT, vínculo Steam
│   ├── Matchmaking.Api/           # filas Redis, MMR Glicko-2
│   ├── Inventory.Api/             # skins, progressão, battle pass
│   ├── Stats.Worker/              # consome eventos de partida, grava histórico
│   └── Shared/                    # contratos, domain, EF Core
├── web/                           # Next.js — site, perfil, leaderboard, loja
├── docker-compose.yml             # Postgres + Redis + serviços (dev local)
└── k8s/                           # Manifests Agones (orquestração de game servers)
```

## Modos de jogo

- **Procurar e Destruir (ranqueado):** 5v5, MR12 (vence quem fizer 13 rounds), economia por round, plant/defuse.
- **Confronto entre Equipes (TDM):** respawn, placar por kills, arenas pequenas, casual.
- **C.1.S. — Cada Um Por Si (FFA):** 8–10 jogadores, respawn 3s, loadout livre, primeiro a 30 kills ou 10 min.
  - **Spawn inteligente:** lista candidatos → descarta os com linha de visão para inimigos → pontua por distância e combate recente (penaliza kills nos últimos ~5s) → proteção de 2s que quebra ao atirar.

## Mapas

14 no total, **12 únicos de produção de arte** (2 reaproveitados entre modos).

### Procurar e Destruir (6)
| Mapa | Tipo | Destaque |
|---|---|---|
| Viaduto | Híbrido | Baixos de viaduto paulistano; oficina (A) e galpão de feira (B) |
| Estaleiro | Híbrido | Porto de contêineres em Santos; verticalidade de guindaste |
| Represa | Aberto | Usina hidrelétrica; casa de máquinas (A, fechado) e vertedouro (B, aberto) |
| Mercadão | Fechado | Mercado municipal; açougue (A) e floricultura (B); vitrais como landmark |
| Garimpo | Aberto | Mina a céu aberto no cerrado; níveis escalonados + túneis |
| Teleférico | Fechado | Morro carioca; cabine cruzando o Meio em ciclo fixo de 25s (cobertura móvel) |

### Confronto entre Equipes (4)
Terminal rodoviário · Arena de society coberta · Pátio de carnaval · Data center.

### C.1.S. (4)
Mercadão e Garimpo reaproveitados (mesmo `.umap`, GameMode e spawns próprios — fluxo circular, sem sightlines longas) + **Metrô** e **Feira noturna** exclusivos (fluxo em "8", ~40% do tamanho de um mapa S&D).

## Arsenal (27 armas)

Todas disponíveis para os dois lados. Dano corpo/cabeça a 0–20m contra colete; cabeça ≈ 3× corpo (pistolas pesadas 4×). TTK levemente menor que CS2 (filosofia CF — AK mata com 3 no peito, igual ao jogo original).

| Arma | Preço | Papel | Dano C/Cb | RPM |
|---|---|---|---|---|
| Glock-18 | spawn | Loadout, pente 20 | 28/84 | 400 |
| USP | spawn | Loadout, silenciada | 34/102 | 352 |
| Tec-9 | $500 | Eco agressiva | 32/96 | 500 |
| Colt M1911 | $600 | One-tap sem capacete | 40/120 | 300 |
| Desert Eagle | $750 | One-tap com capacete | 56/224 | 267 |
| Uzi | $1.000 | Spray barato | 25/75 | 820 |
| Remington 870 | $1.100 | Pump de eco | 9×25 | — |
| MP5 | $1.350 | SMG versátil | 28/84 | 750 |
| SPAS-12 | $1.450 | Híbrida pump/semi | 9×28 | — |
| MP7 | $1.600 | Mobilidade máxima | 30/90 | 780 |
| M700 | $1.700 | Bolt leve (scout) | 76/300 | — |
| Saiga-12 | $1.850 | Semi com pente destacável | 8×22 | — |
| Galil AR | $1.900 | Rifle de eco (spray) | 31/93 | 666 |
| FAMAS | $1.900 | Rifle de eco (burst opcional) | 30/90 | 666 |
| XM1014 | $2.050 | Semi de cadência alta | 8×20 | — |
| M16 | $2.200 | Burst fixo de 3 | 34/102 | burst |
| P90 | $2.350 | Anti-rush, pente 50 | 25/70 | 857 |
| AK-47 | $2.900 | One-tap cabeça, DPS alto | 37/111 | 600 |
| M4A1 | $2.900 | Silenciada, spray controlável, **wallbang 100%** | 34/97 | 666 |
| RPK | $2.900 | Spray sustentado, pente 75 | 33/99 | 600 |
| SG 553 | $3.100 | Mira 2x | 35/105 | 545 |
| AUG | $3.100 | Mira 1.5x | 34/99 | 600 |
| HK417 | $3.200 | Battle rifle de tap (fiel ao CF): dano > AK, estabilíssima em tiro único, fraca no close pela cadência; pente 20; 2 no peito + 1 qualquer = kill | 42/126 | 450 |
| PSG-1 | $4.300 | Sniper semi-automática | 72/216 | 240 |
| AWP | $4.750 | One-shot no torso | 115/410 | — |
| M60 | $4.800 | Supressão, pente 100 | 35/91 | 535 |
| Barrett M82A1 | $5.500 | Anti-material: atravessa cobertura leve e alvenaria | 122/450 | — |

**Corpo a corpo:** faca padrão + machado/katana/kukri como variantes cosméticas (stats idênticos).

## Sistema de recoil

`CFRecoilComponent` (C++, roda no cliente dono):
- **Vertical determinístico** via `UCurveFloat` por arma, indexado pelo número do tiro — sobe rápido nos tiros 1–5 e estabiliza num teto (spray CF "sobe e fica", sem padrão lateral decorado).
- **Horizontal:** jitter aleatório pequeno dentro de cone limitado, crescendo 5% por tiro com teto de 2×.
- **Recuperação automática** ao parar de atirar, só do que não foi compensado manualmente.
- **Servidor revalida** o spread com seed sincronizada — cliente nunca decide hit.

Todos os stats vêm de `Config/Weapons/DT_Weapons.csv` via DataTable (struct `FWeaponStats`). Balancear = editar planilha; no futuro, hot-reload no servidor entre partidas.

## Modelo de dano e penetração

### Penetração (hierarquia de materiais)
| Material | Arma comum | M4A1 | Barrett M82A1 |
|---|---|---|---|
| Madeira / zinco / lona | 70% do dano passa | 100% | 100% |
| Alvenaria / tijolo | 30% | 30% | 100% (trata como madeira) |
| Concreto / aço | 0% | 0% | 0% |

> A rivalidade AK vs M4, fiel ao CF de torneio: **AK = one-tap na cabeça; M4A1 = bala varada (wallbang 100%)**. Regra de level design: todo bombsite tem ≥1 cobertura de madeira (varável); coberturas principais de defesa são alvenaria.

### Capacete (modelo CrossFire)
O capacete absorve 15% do dano de cabeça, protege **um** headshot não-letal e **voa da cabeça** (feedback visível icônico do CF). O segundo headshot pega cheio. Combinado com o falloff por distância, cada arma tem um alcance de one-tap:

| Arma | One-tap SEM capacete | One-tap COM capacete |
|---|---|---|
| Glock / Uzi / P90 | Nunca (2 tiros) | Nunca |
| USP | Sempre | Não (capacete voa, 2º mata) |
| Colt M1911 | Sempre | Até ~12 m |
| Desert Eagle | Sempre | Sempre |
| M4A1 | Até ~30 m | Não (identidade CF) |
| AK-47 | Sempre | Até ~40 m |
| HK417 | Sempre | Até ~44 m |
| SG 553 / AUG | Sempre | Até ~25 m |
| M700 / PSG-1 / AWP / Barrett | Sempre | Sempre |

> Cria a decisão de compra que o CS não tem: capacete vale ouro contra time de M4/SMG, vale menos contra time de AK/HK417.

## Granadas

Mecânica do CS, pancada do CF. 4 slots de utilitário por jogador.

| Granada | Preço | Limite | Spec |
|---|---|---|---|
| HE | $300 | 1 | 110 no epicentro, raio letal 3m, raio total 9m (queda linear). Mais letal que a HE do CS |
| Flash | $200 | 2 | Cegueira até 4,5s conforme ângulo/distância; de costas reduz 70%; pop duplo funciona |
| Smoke | $300 | 1 | **Volumétrica opaca total estilo CS2** (nada da fumaça rala do CF). Dura 18s, vaza por porta/janela; rajada de bala abre túnel momentâneo; HE dentro abre clareira de ~2s |
| Molotov | $450 | 1 | Nega área por 7s, ~80 de dano se atravessar; apaga com smoke |

### Lineups (atalhos de granada por design)
- Cada site tem **3 lineups oficiais de smoke** com landmark de mira construído no cenário (ponta do vitral, gancho do guindaste).
- Mapas fechados: claraboia/vão de granada no teto (o "atalho" CF de jogar bomba por cima sem se expor).
- Mapas abertos: céu liberado, jogo de arco longo.
- Jump-throw nativo e 100% consistente. Modo treino mostra trajetória e ponto de queda.
- Limite saudável: ~10 lineups úteis por mapa. Todo choke smokável sem pixel-perfect.

## Sistema de compra

**Híbrido:**
- **Ranqueada (S&D):** compra por round estilo CS/Valorant — o mais justo e competitivo (a economia É a camada estratégica: eco rounds, force buys, viradas).
- **Casual (TDM e C.1.S.):** até 3 loadouts montados, troca no respawn — feel CrossFire/mochila puro.
- Em todos os modos: escolha da pistola de spawn (Glock ou USP) e skins equipadas via loadout.

## Economia da ranqueada

$800 iniciais · colete + capacete $1.000 · full buy completo ~$4.700 · recompensa por kill por categoria (rifle $300, SMG $600, escopeta $900, AWP/Barrett $100) · bônus de derrota progressivo garantindo full buy após ~3 rounds perdidos.

## Bíblia de mapas

### Métricas do jogador (a régua de onde tudo deriva)
- Cápsula: 1,83m em pé · 1,20m agachado · 0,61m de largura · linha dos olhos 1,70m.
- Velocidades: faca 6,2 m/s · rifle 5,5 · AWP/Barrett 5,0 · agachado 2,8 (15% mais ágil que CS).
- Pulo 0,65m · crouch-jump 0,85m · degrau automático 0,45m.

### Alturas de cobertura (sagradas — nada fora destas)
- **Meia cobertura 1,10m:** protege agachado, expõe a cabeça de quem está em pé (cobertura de duelo).
- **Caixa escalável 1,30m:** sobe com crouch-jump; empilhada vira plataforma de boost.
- **Cobertura cheia 1,90m:** passa da linha dos olhos, esconde por completo.

> Isso elimina o head-glitch aleatório do CrossFire (onde cada caixa tem altura diferente).

### Larguras padrão
Porta 1,2m · corredor secundário 2,5m · choke principal de site 4–5m (cabe 1 smoke, não 2) · praça de bombsite 12–18m · plant zone 4×4m.

### Timings (balanceiam o mapa inteiro)
| Trajeto | Alvo | Regra |
|---|---|---|
| TR spawn → entrada de site | 12–16s | Tempo do CT montar defesa |
| CT spawn → posição de defesa | 6–9s | **CT SEMPRE chega antes** |
| Rotação CT entre sites | 14–20s | Punir rotação adiantada |
| Meio → qualquer site | ≤8s | Por isso o Meio vale a briga |
| Distância física entre sites | 40–60m | Deriva da rotação |
| Bomb timer / defuse | 40s / 10s (5 com kit) | — |

### Endereçamento (callouts)
Cada zona é um `CalloutVolume` que alimenta radar + ping inteligente + killfeed/replay. ID padrão: `SD_MERCADAO/A-03_ACOUGUE` (prefixo de site A/B/M/T/C). Regra de rádio: 1 palavra, ≤3 sílabas, única, gritável. Meta: 18–24 callouts por mapa S&D.

### Interativos (o melhor dos dois jogos)
| Elemento | Regra |
|---|---|
| Porta de madeira quebrável | 150 HP, wallbang antes de cair, barulho = informação. Máx. 2/mapa, nunca em rota principal |
| Janela de vidro | Quebra com 1 tiro; toda janela jogável é quebrável |
| Duto de ventilação | Rota furtiva agachada; som metálico alto ao pisar |
| Escada vertical (ladder) | Assinatura CF; 1–2/mapa; subir guarda a arma |
| Galinhas / pombos | Sensor orgânico de rush (voam e fazem barulho) |
| Elemento móvel em timer | Cabine do Teleférico, ciclo de 25s |
| Penetração de material | 3 níveis (ver tabela de penetração) |

### Pixels de mira e skill spots
Toda esquina de duelo tem marcador visual sutil na altura da cabeça (pixel de mira vira linguagem do mapa, não segredo). Head-glitch proibido em posição primária; permitido como skill spot secundário com counter-ângulo. 1–2 skill jumps por mapa. Smokes a 2,2m de altura (one-way smoke não existe por design).

### Tipologia (pool equilibrado)
| Tipo | Sightlines | Armas favorecidas | Mapas |
|---|---|---|---|
| Aberto | 40m+ | AWP, Barrett, HK417, SG/AUG | Represa, Garimpo |
| Fechado | ≤20m | SMGs, escopetas, AK no susto | Mercadão, Teleférico |
| Híbrido | misto | compra mista | Viaduto, Estaleiro |

Regra estrutural fixa: 3 rotas · todo site com 2 entradas + 1 rota de retake do CT · céu jogável definido por mapa.

## Custos de hospedagem

Estimativa mensal (USD):

| Fase | Servidores de partida | Backend + banco + CDN | Total |
|---|---|---|---|
| Protótipo (LAN/amigos) | $0 (VPS atual) | ~$20 | **~$20** |
| Alfa fechado (~100 jog.) | ~$90 | ~$70 | **~$160** |
| Beta aberto (~500 CCU) | ~$350 | ~$200 | **~$550** |
| Lançamento (~1000 CCU) | ~$1.800 | ~$800 | **~$2.600** |

> Base: bare metal OVH (~$70/máquina, 10–15 partidas simultâneas cada) + autoscaling no lançamento (instâncias spot economizam 50–85%). EOS gratuito. Steam: $100 únicos.

## Roadmap

1. **(3–4 meses)** Gray-box de 1 mapa S&D, 4 armas, servidor dedicado, 5v5 funcional — valida o feel CF+CS2.
2. **(3 meses)** Backend .NET (auth, fila, stats), 2 mapas com arte, alfa fechado.
3. **(4–6 meses)** Ranqueada/MMR, anti-cheat, modos TDM, beta aberto.
4. **Lançamento** Steam F2P, monetização 100% cosmética.

> **Realismo:** a Fase 1 é viável solo. O maior custo escondido é arte de 12 mapas com qualidade CS2 — por isso o roadmap valida a jogabilidade antes de investir pesado em arte.

## Decisões em aberto

- [ ] Nome do jogo e codinome
- [ ] Nomes comerciais das armas (renames legais)
- [ ] Formato competitivo do C.1.S. (entra na ranqueada ou só casual?)
- [ ] Qual dos 6 mapas S&D abre o desenvolvimento (recomendação: Mercadão)
- [ ] Conta GitHub de destino do repositório

---

*Documento de fundação — versão inicial. Construído iterativamente; cada seção pode ser detalhada em profundidade conforme o projeto avança.*
