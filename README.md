# Tensura KubeJS

KubeJS addon for registering ManasCore/Tensura skills and races from startup scripts.

Full documentation: [TensurJS Wiki](https://github.com/AQZL/TensurJS/wiki)

Example registry IDs:

- Skill registry: `manascore:skill`
- Race registry: `manascore:race`

Put scripts in your pack's `kubejs/startup_scripts`.

Notes:

- Startup scripts require a game restart. `/reload` is only for server scripts.
- A registered skill is not automatically learned by the player.
- A skill is not added to the reincarnation/reset-scroll skill pool unless the script calls `.startingSkill(true)` or `.secondStartingSkill(true)`.
- `.startingSkill(false)` removes that skill from the runtime starting skill pool for this launch.
- `ctx.setCooldown(seconds)` uses Tensura's cooldown unit shown by the UI, so `ctx.setCooldown(3)` is a 3-second cooldown.
- Single-mode skills display Tensura's default mode name. Multi-mode skills can define mode names with `.modeName(index, name)` or `.mode(index, id, name, description)`.
- Race evolution requirements are written on the target race. For example, `.epRequirement(1000)` on `example:oni` means evolving into `example:oni` needs 1000 base EP.
- Tensura's EP evolution requirement checks base EP: base max aura + base max magicule.

## Modify existing Tensura skills

Call `TensuraKubeJS.modifySkill(id)` from a startup script. Every configured callback replaces the original callback with the same name.

```js
TensuraKubeJS.modifySkill('tensura:flame_manipulation')
  .icon('example:skill/flame_manipulation')
  .cooldown(5)       // default for every mode, in seconds
  .cooldown(0, 3)    // mode 0
  .cooldown(1, 8)    // mode 1
  .canActivate(ctx => true)
  .onPressed(ctx => {
    if (ctx.cooldown() > 0) return
    ctx.spawnEntity('minecraft:villager')
    ctx.setCooldown(3)
  })
```

The icon above is loaded from `kubejs/assets/example/textures/skill/flame_manipulation.png`. Supported replacement callbacks include activation checks, toggle/tick/press/hold/release/scroll, learn/forget/master, damage hooks, and death.

## Grant elemental spirits

Spirit helpers can be called from any server script condition:

```js
if (!TensuraKubeJS.hasSpiritAtLeast(player, 'fire', 'greater')) {
  TensuraKubeJS.grantSpirit(player, 'fire', 'greater')
}
```

`grantSpirit` follows Tensura's successful prayer rewards: it upgrades the stored spirit level, grants every matching spiritual magic up to that tier, and grants the element's manipulation skill. Duplicate or lower-tier grants return `false`. Levels are `lesser`, `medium`, `greater`, and `lord`. Elements are `darkness`, `earth`, `fire`, `light`, `space`, `water`, `wind`, and `holy`. Chinese aliases are also accepted. See `example_scripts/server_scripts/tensura_kubejs_spirits.js`.

For worlds that used an older version of this addon, call `TensuraKubeJS.syncSpiritRewards(player)` on login to grant missing magic for spirits already stored on the player.

```js
StartupEvents.registry(TensuraKubeJS.SKILL_REGISTRY, event => {
  event.create('example:surging_aura')
    .name('Surging Aura')
    .type('extra')
    .description('A simple toggled aura created from KubeJS.')
    .startingSkill(false)
    .modes(2)
    .mode(0, 'pulse', 'Pulse', 'Heal yourself while toggled.')
    .mode(1, 'burst', 'Burst', 'A placeholder second mode.')
    .maxMastery(200)
    .canBeToggled(true)
    .canTick(ctx => ctx.isToggled())
    .heldAttribute('minecraft:generic.movement_speed', 'surging_aura_speed', 0.15, 'ADD_MULTIPLIED_TOTAL')
    .onPressed(ctx => {
      ctx.setCooldown(3)
    })
    .onTick(ctx => {
      if (ctx.entity.tickCount % 40 === 0) {
        ctx.entity.heal(1)
      }
    })
})

StartupEvents.registry(TensuraKubeJS.RACE_REGISTRY, event => {
  event.create('example:lesser_oni')
    .name('Lesser Oni')
    .description('A KubeJS-created race with an evolution route.')
    .difficulty('intermediate')
    .baseAura(500, 600)
    .baseMagicule(500, 600)
    .maxHealth(8)
    .maxSpiritualHealth(40)
    .movementSpeed(0.03)
    .attribute('minecraft:generic.max_health', 'lesser_oni_health', 8, 'ADD_VALUE')
    .intrinsicSkill('example:surging_aura')
    .defaultEvolution('example:oni')

  event.create('example:oni')
    .name('Oni')
    .difficulty('hard')
    .previousEvolution('example:lesser_oni')
    .baseAura(2000, 2500)
    .baseMagicule(2000, 2500)
    .epRequirement(1000)
    .attribute('minecraft:generic.attack_damage', 'oni_damage', 0.25, 'ADD_MULTIPLIED_TOTAL')
})
```

The build scans `lib/`, `../tensura_ablazeaqzl/lib`, and `../server1.21/lib` for local ManasCore/Tensura jars.
