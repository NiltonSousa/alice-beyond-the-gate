# Alice Além do Portão — Design do Vertical Slice

**Data**: 2026-09-23
**Status**: Aprovado para implementação (aguardando escrita do plano via `writing-plans`)

## Visão geral

Um RPG 2D indie estrelado por gatos, narrativamente inspirado (não copiado) em
*Sea of Stars*. Duas casas vizinhas, cada uma com um grupo de gatos pertencente
a um tutor humano diferente. Os gatos sempre puderam se comunicar entre si —
isso nunca foi segredo entre eles, só para os humanos. Os dois tutores, Diana e
Nuno (namorados, moram perto um do outro), desaparecem misteriosamente. Comida
e água acabam. Os gatos investigam e entram em contato com Alice, uma gata da
casa da Diana já falecida, agora no mundo espiritual, que se torna guia do
grupo — mas é limitada por regras reais do universo sobre o que os mortos
podem revelar aos vivos.

Esta spec cobre o **vertical slice**: uma fatia vertical de ~20-30 minutos de
jogo, cobrindo do início da história até a primeira pista concreta sobre o
desaparecimento.

## Premissa e elenco

### Humanos

- **Diana** e **Nuno** são namorados e moram em casas vizinhas. Os grupos de
  gatos das duas casas já se conhecem antes do jogo começar — não há jornada
  narrativa necessária para uni-los.
- Os dois desaparecem no início da história. Se sumiram juntos, pela mesma
  causa, ou é coincidência, fica em aberto — é material para o Ato IV (A
  VERDADE), não para o vertical slice.

### Elenco felino

**Casa de Diana** (7 gatas, incluindo Alice):

| Gata | Aparência | Personalidade / traço narrativo |
|---|---|---|
| Alice | Cinza com branco, olhos laranjas | Já falecida; guia espiritual. Amava comida (churu/sachê) — humaniza sua caracterização emocional. |
| Bela | Ciamesa puxada pro marrom, olhos azuis "galácticos" | A mais velha, mal-humorada, briga direto com o Stopa. |
| Stopa | Frajola, único macho do grupo | Se estica muito; briga com a Bela. |
| Luna | Preta, desengonçada mas esperta | Nome sugere tema lunar. |
| Amora | Tricolor, rabo estilo Pikachu | Possível tema elétrico. |
| Maria Fafa | Tricolor puxada pro branco, parece pandinha | Muito observadora, atenta a detalhes. |
| Eva | Branca com toques bege, cauda de raposa | Possível tema de fogo. |

**Casa de Nuno** (2 gatas):

| Gata | Aparência | Personalidade / traço narrativo |
|---|---|---|
| Gaya | Preta, olhos verdes | Dá cambalhotas, esperta. |
| Moana | Rajada (preto e laranja), olhos verdes | Bem jovem, ainda sem características marcantes — isso é intencional na narrativa. |

### Poderes

Os poderes especiais são **mágicos e reais no mundo do jogo**, não estilização
de comportamento felino comum. A origem e o motivo de só alguns gatos terem
poderes é um mistério a se revelar gradualmente nas escalas MUNDO DOS GATOS /
MUNDO ESPIRITUAL — possivelmente ligado à mesma natureza que permite à Alice
se comunicar com os vivos.

### Elenco jogável e progressão

- **Início (vertical slice)**: 4 gatos jogáveis — **Bela + Stopa** (Diana) e
  **Gaya + Moana** (Nuno), únicos dois gatos daquela casa.
- **Progressão pós-slice**: a cada ato/capítulo concluído na campanha
  principal, +2 gatos são desbloqueados (4 → 6 → 8), puxados do restante do
  elenco da Diana (Luna, Amora, Maria Fafa, Eva). Isso é progressão de
  história, não recompensa de New Game+.

| Gato | Casa | Traço de exploração | Habilidade de combate |
|---|---|---|---|
| Bela | Diana | Sensível (sente presenças espirituais) | Olhar paralisante |
| Stopa | Diana | Ágil (alcança lugares altos) | Alcance/elasticidade |
| Gaya | Nuno | Curiosa (acha objetos escondidos) | A definir em implementação |
| Moana | Nuno | Pequena (cabe em espaços apertados) | A definir em implementação |

O traço "sensível" da Bela reforça sua ligação simbólica com Alice e o mundo
espiritual desde a primeira casa jogável.

## Engine e stack

**Godot 4 + GDScript** — confirmado após comparação de ferramentas (Godot,
Phaser, Unity, RPG Maker) para o perfil do desenvolvedor: programador
experiente, primeira vez em jogos, preferência por abordagem code-first.

## Arquitetura de dados: Abordagem C (dados + strategy pattern)

Conteúdo (stats, sprites, textos, diálogo) vive em **Resources** (`.tres`) —
dado puro, sem lógica. Comportamento único por habilidade especial vive em
**pequenos scripts plugáveis** que implementam um contrato comum
(`AbilityEffect`), equivalente ao *strategy pattern* já familiar em
desenvolvimento de software tradicional. O sistema de combate nunca contém
`if personagem == "Bela"` — ele só invoca `character.special_ability.execute(...)`
e deixa o módulo certo cuidar do efeito.

Essa escolha foi preferida a:
- **Dado puro para tudo** (Abordagem A): não dá conta de efeitos muito
  específicos (paralisia, alcance) sem generalizar demais.
- **Uma cena/classe por personagem** (Abordagem B): cria 8+ cenas
  quase-duplicadas conforme o elenco cresce, dificultando balanceamento.

### Modelo de dados

```gdscript
# scripts/characters/cat_character.gd
class_name CatCharacter
extends Resource

@export var display_name: String
@export var portrait: Texture2D
@export var sprite_frames: SpriteFrames

@export var max_hp: int
@export var base_attack: int
@export var base_defense: int

@export var special_ability: AbilityEffect
@export var exploration_trait: String  # "pequena" | "ágil" | "curiosa" | "forte" | "sensível"
```

```gdscript
# scripts/abilities/ability_effect.gd
class_name AbilityEffect
extends Resource

@export var ability_name: String
@export var description: String
@export var mp_cost: int

func execute(user: BattleActor, target: BattleActor, battle: BattleContext) -> void:
    pass  # sobrescrito por cada habilidade concreta
```

Exemplo concreto (olhar paralisante da Bela):

```gdscript
# scripts/abilities/paralyze_gaze.gd
class_name ParalyzeGaze
extends AbilityEffect

func execute(user: BattleActor, target: BattleActor, battle: BattleContext) -> void:
    target.apply_status("paralyzed", duration_turns=1)
    battle.play_vfx("paralyze_flash", target.position)
```

## Estrutura de pastas

```
alice-beyond-the-gate/
├── project.godot
├── scenes/
│   ├── characters/
│   │   └── cat_character.tscn
│   ├── maps/
│   │   ├── casa_diana/
│   │   ├── casa_nuno/
│   │   └── vizinhanca/
│   ├── battle/
│   │   └── battle_scene.tscn
│   └── ui/
│       ├── dialogue_box.tscn
│       └── hud.tscn
├── scripts/
│   ├── characters/
│   │   └── cat_character.gd
│   ├── abilities/
│   │   ├── ability_effect.gd
│   │   ├── paralyze_gaze.gd
│   │   └── stretch_reach.gd
│   ├── battle/
│   │   └── battle_state_machine.gd
│   ├── dialogue/
│   │   └── dialogue_runner.gd
│   └── autoload/
│       └── game_state.gd
├── resources/
│   ├── characters/
│   │   ├── bela.tres
│   │   ├── stopa.tres
│   │   ├── gaya.tres
│   │   └── moana.tres
│   ├── abilities/
│   │   ├── paralyze_gaze.tres
│   │   └── stretch_reach.tres
│   └── dialogue/
│       └── casa_diana_intro.tres
├── assets/
│   ├── sprites/
│   ├── tilesets/
│   └── audio/
└── docs/
    └── superpowers/
        └── specs/
```

`autoload/game_state.gd` é um singleton global (autoload do Godot) que guarda
o que persiste entre cenas: elenco desbloqueado, progresso de atos, flags de
história.

## Exploração

Movimento 2D com colisão contra o cenário. Câmera segue o gato ativo
(follow loosely com deadzone), presa aos limites do mapa.

O mundo é modelado na escala do gato, não do humano — um sofá é uma área
explorável, uma caixa é um esconderijo, o vão embaixo de um móvel é uma
passagem. Isso é implementado via **camadas de colisão** (collision
layers/masks): um objeto do mapa só é passável por quem tem o traço de
exploração certo, verificado de forma genérica pelo traço (`exploration_trait`)
do personagem ativo — nunca por checagem nomeada a um gato específico.

Interação com objetos do mundo (caixa, gaveta, baú) usa um componente
genérico `Interactable` com um sinal (`interacted`); o que está "plugado" a
esse sinal (abrir diálogo, revelar item, abrir passagem) varia por instância,
mas o sistema é único.

## Sistema de diálogo

Cada `DialogueResource` (`.tres`) contém uma sequência de falas (speaker,
texto, ramificações opcionais). Um `DialogueRunner` interpreta esses dados e
expõe sinais (`line_shown`, `choice_made`, `dialogue_ended`) para o resto do
jogo reagir.

### Alice e as regras do mundo espiritual

Alice tem cena própria (`alice_spirit.tscn`), reaproveitando a sprite base de
gato com um efeito visual distinto (semi-transparência, partícula sutil) —
reconhecível como "ainda a Alice", mas marcada como espiritual, alinhado à
diretriz de arte de uma linguagem visual distinta porém unificada para o
mundo espiritual.

A regra de que Alice **não pode revelar tudo aos vivos** é modelada como
**flags de progresso narrativo** em `game_state.gd` (ex: `has_met_alice`,
`knows_humans_alive`), que gateiam quais falas do `DialogueResource` dela
estão disponíveis. A limitação não é arbitrária no código — ela reflete uma
regra real do universo do jogo de forma mecânica: as falas além do permitido
simplesmente não existem no dialogue tree ainda.

No vertical slice, o primeiro contato acontece após a primeira investigação;
Alice confirma apenas "eles estão vivos" e "eles precisam de vocês".

## Combate

Sistema de estados, validando o loop mais simples possível antes de qualquer
complexidade adicional (fraquezas elementais, combos, timing de input ficam
para depois do vertical slice):

```
BATTLE_START
    ↓
SELECT_ACTION  ←──────────────┐
    ↓ (ator escolhe: atacar / habilidade / defender)
RESOLVE_ACTION                │
    ↓                         │
CHECK_END ──(ninguém morreu)──┘  (próximo ator na ordem de turno)
    ↓ (condição de vitória/derrota atingida)
BATTLE_END (vitória → recompensa | derrota → retry)
```

- **SELECT_ACTION**: jogador escolhe para o ator da vez (um gato por vez);
  inimigo decide via IA simples (ex: atacar o gato com menor HP, ou alvo
  aleatório no vertical slice).
- **RESOLVE_ACTION**: ataque básico é cálculo puro
  (`base_attack - target.base_defense`); habilidade especial delega para
  `special_ability.execute(...)`.
- **CHECK_END**: verifica HP de todos após cada ação; decide continuar ou
  encerrar a batalha.
- **Ordem de turno**: fixa no vertical slice (todos os aliados agem, depois
  todos os inimigos) — iniciativa/velocidade individual é uma extensão
  futura, não necessária para o loop mínimo.

**Escopo de combate do vertical slice**: 1 tipo de inimigo comum, 1
mini-boss, ataque básico, defesa, 1 habilidade especial por gato, HP,
vitória/derrota.

## Mapas do vertical slice

1. **Casa A (Diana)** — tutorial de movimento/interação com Bela+Stopa.
2. **Casa B (Nuno)** — mesmo tutorial reforçado, com Gaya+Moana, usando os
   traços "curiosa" e "pequena".
3. **Área externa pequena (vizinhança)** — os dois grupos se encontram; a
   partir daqui o jogador alterna livremente entre os 4 gatos.
4. **Área de exploração pequena** — leva à primeira pista concreta.
5. **Pequena dungeon** — primeiro combate e o mini-boss.

## Roadmap de milestones

Ordem seguindo a diretriz do prompt mestre original (jogabilidade > conteúdo
> polimento; validar cada sistema no seu nível mais simples antes de
complexidade adicional):

1. **Fundação** — projeto Godot criado, estrutura de pastas, 1 gato
   controlável (arte placeholder) com movimento + colisão + câmera na Casa A.
2. **Interação e diálogo** — `Interactable` genérico, `DialogueRunner`
   funcional, uma conversa de teste.
3. **Segundo gato + traços** — Stopa jogável, alternância entre os 2, traço
   "ágil" funcionando como mecânica real de exploração.
4. **Combate mínimo** — `BattleStateMachine` completo (ataque básico, HP,
   vitória/derrota), sem habilidades especiais ainda.
5. **Habilidades especiais** — `AbilityEffect` + paralisar (Bela) e
   alcance (Stopa) plugados no combate.
6. **Casa B + Gaya/Moana** — replica milestones 1-3 para a segunda casa e o
   segundo par de gatos.
7. **Alice + narrativa** — primeiro contato, flags de progresso, diálogo
   gateado.
8. **Conectar tudo** — vizinhança, área de exploração, dungeon, mini-boss,
   primeira pista concreta — fechando o vertical slice completo.

O primeiro milestone concreto a ser detalhado em plano de implementação é o
**Milestone 1 — Fundação**.

## Fora de escopo para o vertical slice

Explicitamente adiado, conforme o prompt mestre e as seções acima:

- Fraquezas elementais, combos cooperativos, efeitos de status complexos,
  timing de input em combate.
- Iniciativa/velocidade individual na ordem de turno.
- Habilidades especiais de Gaya e Moana (a definir durante implementação do
  Milestone 6).
- Qualquer gato além dos 4 iniciais (Luna, Amora, Maria Fafa, Eva) — entram
  na progressão pós-slice.
- Resolução de qualquer parte do mistério além da primeira pista concreta —
  a verdade sobre Diana e Nuno, a natureza dos poderes, e a história pessoal
  de Alice pertencem a atos posteriores.
- Arte final em pixel art no nível Sea of Stars — o vertical slice pode usar
  placeholders/asset packs licenciados; a ambição visual completa é de longo
  prazo.
