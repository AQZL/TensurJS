# TensurJS

TensurJS is a KubeJS addon for Minecraft 1.21.1 NeoForge that exposes selected ManasCore and Tensura systems to scripts.

## Features

- Register custom Tensura skills from startup scripts.
- Register custom races, attributes, abilities, and evolution routes.
- Add races and skills to reincarnation starting pools.
- Replace icons, cooldowns, activation checks, and behavior of existing skills.
- Grant elemental spirits with the same magic rewards as a successful Tensura prayer.

## Documentation

- [Installation](https://github.com/AQZL/TensurJS/wiki/Installation)
- [Creating Skills](https://github.com/AQZL/TensurJS/wiki/Creating-Skills)
- [Creating Races](https://github.com/AQZL/TensurJS/wiki/Creating-Races)
- [Modifying Existing Skills](https://github.com/AQZL/TensurJS/wiki/Modifying-Existing-Skills)
- [Elemental Spirits](https://github.com/AQZL/TensurJS/wiki/Elemental-Spirits)
- [Context Reference](https://github.com/AQZL/TensurJS/wiki/Context-Reference)

## Requirements

- Minecraft 1.21.1
- NeoForge 21.1+
- KubeJS 2101.7
- Architectury API 13
- ManasCore 4 modules required by Tensura
- Tensura 2.0

Registry IDs exposed to KubeJS:

- Skills: `TensuraKubeJS.SKILL_REGISTRY` (`manascore:skill`)
- Races: `TensuraKubeJS.RACE_REGISTRY` (`manascore:race`)

Put registry and skill override scripts in `kubejs/startup_scripts/`. Startup script changes require a full game restart; `/reload` cannot recreate registries.

## Create a Custom Skill

```js
StartupEvents.registry(TensuraKubeJS.SKILL_REGISTRY, event => {
  event.create('example:surging_aura')
    .name('Surging Aura')
    .description('A two-mode toggled aura.')
    .type('extra')
    .icon('example:skill/surging_aura')
    .startingSkill(false)
    .modes(2)
    .mode(0, 'pulse', 'Pulse', 'A short pulse mode.')
    .mode(1, 'burst', 'Burst', 'A stronger burst mode.')
    .maxMastery(200)
    .canBeToggled(true)
    .canTick(ctx => ctx.isToggled())
    .heldAttribute(
      'minecraft:generic.movement_speed',
      'surging_aura_speed',
      0.15,
      'ADD_MULTIPLIED_TOTAL'
    )
    .onPressed(ctx => {
      ctx.setCooldown(ctx.mode === 0 ? 3 : 8)
    })
    .onTick(ctx => {
      if (ctx.entity.tickCount % 40 === 0) {
        ctx.entity.heal(1)
      }
    })
})
```

The icon above is loaded from:

```text
kubejs/assets/example/textures/skill/surging_aura.png
```

Registered skills are not automatically learned. Use `.startingSkill(true)` or `.secondStartingSkill(true)` only when the skill should enter a reincarnation skill pool.

## Create a Custom Race and Evolution Route

Evolution requirements belong to the target race. In this example, `example:oni` requires 1000 base EP because `.epRequirement(1000)` is declared on Oni.

```js
StartupEvents.registry(TensuraKubeJS.RACE_REGISTRY, event => {
  event.create('example:lesser_oni')
    .name('Lesser Oni')
    .description('The first stage of an example evolution route.')
    .difficulty('intermediate')
    .startingRace(true)
    .randomStartingRace(true)
    .baseAura(500, 600)
    .baseMagicule(500, 600)
    .maxHealth(8)
    .maxSpiritualHealth(40)
    .attack(2)
    .movementSpeed(0.03)
    .attribute(
      'minecraft:generic.max_health',
      'lesser_oni_health',
      8,
      'ADD_VALUE'
    )
    .intrinsicSkill('example:surging_aura')
    .defaultEvolution('example:oni')

  event.create('example:oni')
    .name('Oni')
    .description('An evolved KubeJS-created race.')
    .difficulty('hard')
    .previousEvolution('example:lesser_oni')
    .baseAura(2000, 2500)
    .baseMagicule(2000, 2500)
    .maxHealth(16)
    .maxSpiritualHealth(80)
    .attack(4)
    .movementSpeed(0.06)
    .epRequirement(1000)
})
```

Tensura evaluates this EP requirement from base maximum aura plus base maximum magicule.

Additional race requirement helpers include:

- `itemConsumeRequirement(item, count, weight?)`
- `itemCarryingRequirement(item, count, weight?)`
- `skillRequirement(skill, mastered, weight?)`
- `evolution(race, progress)`
- `evolutionProgress(callback)`

## Modify an Existing Skill

Each configured callback replaces the original callback with the same name. Do not define `onPressed` when only the icon or cooldown should change.

```js
TensuraKubeJS.modifySkill('tensura:magic_sense')
  .icon('example:skill/custom_magic_sense')
  .cooldown(5)
```

Multi-mode cooldowns are zero-based:

```js
TensuraKubeJS.modifySkill('tensura:some_multi_mode_skill')
  .cooldown(10)
  .cooldown(0, 3)
  .cooldown(1, 8)
```

## Grant Elemental Spirits

Spirit conditions belong in `kubejs/server_scripts/`.

```js
PlayerEvents.tick(event => {
  const player = event.player

  if (player.experienceLevel >= 60 && !TensuraKubeJS.hasSpiritAtLeast(player, 'fire', 'greater')) {
    TensuraKubeJS.grantSpirit(player, 'fire', 'greater')
  }
})
```

`grantSpirit` upgrades the stored spirit tier, triggers summon rewards, grants all matching spiritual magic up to that tier, and grants the element's manipulation skill.

For worlds created with an older TensurJS build:

```js
PlayerEvents.loggedIn(event => {
  TensuraKubeJS.syncSpiritRewards(event.player)
})
```

## Building from Source

Third-party mod jars are intentionally excluded from this repository. Put compatible KubeJS, Rhino, Tensura, ManasCore, and required dependency jars in `lib/`, then run:

```powershell
.\gradlew.bat build
```

The output jar is created in `build/libs/`.
