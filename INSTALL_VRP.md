# rgarage_nitro — Installation on vRP

Also covers **Creative Network** and **Creative V5**, which use this same bridge.
For VRPeX see `INSTALL_VRPEX.md`.

---

## 1. Dependencies

Start these **before** `rgarage_nitro` in `server.cfg`:

- `vrp`
- `oxmysql`
- `ox_lib`

```cfg
ensure vrp
ensure oxmysql
ensure ox_lib
ensure rgarage_nitro
```

## 2. Database

Run `rgarage_nitro.sql` once. It creates two tables:

- `rgarage_nitro` — nitro type, amount, colour and bottle position, per plate
- `rgarage_nitro_player_prefs` — pressure and HUD layout, per player

Missing columns are added automatically on start, so upgrading from an older
version needs nothing else.

> Requires **MySQL 5.7+** or **MariaDB 10.2+**.

## 3. config.lua

```lua
Config.Framework = "vrp"      -- also valid: "creative_network", "creative_v5"
Config.Language  = "pt_br"    -- "en", "pt_br", "es", "fr", "de", "it", "ru", "zh", "ja", "ko"
```

## 4. fxmanifest.lua — required for vRP

Two lines must be **uncommented**:

```lua
shared_scripts {
    'config.lua',
    'Locales.lua',
    '@ox_lib/init.lua',
    '@vrp/lib/utils.lua'      -- <= REQUIRED on vRP
}

dependencies {
    'ox_lib',
    'oxmysql',
    'vrp'                      -- <= REQUIRED on vRP
}
```

Without `@vrp/lib/utils.lua` there is no `Proxy`, the bridge cannot reach vRP, and
using the item does nothing at all.

> **Linux servers are case sensitive.** Some vRP builds ship `lib/Utils.lua` with a
> capital U. Check the real filename in your `vrp/lib/` folder and match it exactly,
> or the resource will not start.

## 5. The item

The resource ships with two items: **`nitrosingle`** (200) and **`nitroduo`** (400).
Names are defined in `Config.Items`.

### 5.1 Add them to your itemlist

In your vRP itemlist (usually `vrp/cfg/items.lua` or your base's equivalent):

```lua
["nitrosingle"] = { "Nitro Single", "Instala nitro simples no veículo.", 0.5, ... },
["nitroduo"]    = { "Nitro Duplo",  "Instala nitro duplo no veículo.",   0.5, ... },
```

Keep the exact names `nitrosingle` and `nitroduo`, or change `Config.Items` to match
whatever you used.

### 5.2 Make the item usable

Where your itemlist handles the "use" action, fire the client event. There are two
items, so there are two calls:

```lua
-- Single nitro (200)
TriggerClientEvent('rgarage_nitro:client:useItem', source, 'nitrosingle')

-- Double nitro (400)
TriggerClientEvent('rgarage_nitro:client:useItem', source, 'nitroduo')
```

On most vRP bases that goes in the item's `choices` / `use` function — one block per
item:

```lua
["nitrosingle"] = {
    choices = {
        ["Usar"] = { function(player, choice)
            local user_id = vRP.getUserId(player)
            if not user_id then return end
            TriggerClientEvent('rgarage_nitro:client:useItem', player, 'nitrosingle')
        end }
    }
},

["nitroduo"] = {
    choices = {
        ["Usar"] = { function(player, choice)
            local user_id = vRP.getUserId(player)
            if not user_id then return end
            TriggerClientEvent('rgarage_nitro:client:useItem', player, 'nitroduo')
        end }
    }
}
```

> **Only want one of them?** Add just the item you want to your itemlist and register
> only that one. The resource works fine with a single item, and nothing else needs
> changing.

> **Do not consume the item yourself.** The resource takes it only after the install
> progress finishes, so a cancelled install never costs the player anything.

## 6. Permissions — `config.lua`

```lua
Config.Commands = {
    enabled    = true,
    nameSingle = "nitrosingle",   -- /nitrosingle
    nameDuo    = "nitroduo",      -- /nitroduo
    groups     = {}               -- empty = anyone
}
```

To restrict, use the **exact** group or permission name from your `Groups.lua`:

```lua
groups = {
    ["Mechanic"] = 0,
    ["Admin"]    = 0,
}
```

The vRP bridge accepts both groups and permissions, including dotted ones
(`"admin.kick"`).

## 7. Notify and progress bar

```lua
Config.Notify   = { enabled = true, type = "ryu" }     -- "ox_lib" | "ryu" | "custom"
Config.Progress = { enabled = true, type = "event" }   -- "event" | "ox_lib" | "custom"
```

- `"ryu"` fires `ryu_hud:notification` (Ryu-Hud)
- `"event"` fires `TriggerEvent("Progress", duration)`, the usual vRP pattern
- `"custom"` calls your own function, set in `CustomFunction`

## 8. Nitro bottle prop (optional)

A physical bottle can be shown on the car. The player places it with a gizmo when
installing, and the spot is saved per plate.

```lua
Config.Prop = {
    enabled        = true,
    placeOnInstall = true,   -- position the bottle before the install starts
    withCollision  = false,  -- keep false while attached to a vehicle
}
```

Both helper commands ship **disabled**. Enable them in `config.lua` only if you need
them:

```lua
Config.Prop.testCommand = "nitroproptest"   -- debug: spawns the model, prints its size to F8
Config.Prop.editCommand = "nitrobottle"     -- lets players re-open the editor after installing
```

With `placeOnInstall = true` the player already positions the bottle during the
install, so neither command is required for normal use.

---

## Troubleshooting

**Using the item does nothing, no error**
The console prints `GetUserId returned nil` when the framework is wrong. Confirm
`Config.Framework = "vrp"` and that `@vrp/lib/utils.lua` is uncommented.

**Resource does not start**
Usually the `utils.lua` path: wrong case on Linux, or `vrp` not started before it.

**The nitro does not save**
Check your MySQL version (5.7+ / MariaDB 10.2+) and that the plate is not blank —
a vehicle with no plate cannot be told apart from any other plateless vehicle.
