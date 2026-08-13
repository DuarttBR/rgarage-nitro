# rgarage_nitro — Installation on QBCore

Covers both **qb-inventory** and **ox_inventory**.

---

## 1. Dependencies

Start these **before** `rgarage_nitro` in `server.cfg`:

- `qb-core`
- `oxmysql`
- `ox_lib`

```cfg
ensure qb-core
ensure oxmysql
ensure ox_lib
ensure rgarage_nitro
```

## 2. Database

Run `rgarage_nitro.sql` once. It creates two tables:

- `rgarage_nitro` — nitro type, amount, colour and bottle position, per plate
- `rgarage_nitro_player_prefs` — pressure and HUD layout, per player

Missing columns are added automatically on start.

> Requires **MySQL 5.7+** or **MariaDB 10.2+**.

## 3. config.lua

```lua
Config.Framework = "qb"     -- "qbcore" also accepted
Config.Language  = "en"
```

## 4. fxmanifest.lua — REQUIRED on QBCore

Two lines must be **commented out**. There is no `vrp` resource on your server, so
leaving them active makes the resource fail to start:

```lua
shared_scripts {
    'config.lua',
    'Locales.lua',
    '@ox_lib/init.lua',
    -- '@vrp/lib/utils.lua'   -- <= MUST stay commented on QBCore
}

dependencies {
    'ox_lib',
    'oxmysql',
    -- 'vrp'                   -- <= MUST stay commented on QBCore
}
```

This is the single most common reason the resource does not boot on QB.

## 5. The item

Two items ship with the resource: **`nitrosingle`** (200) and **`nitroduo`** (400),
defined in `Config.Items`.

### Option A — ox_inventory

Open `ox_inventory/data/items.lua` and add:

```lua
['nitrosingle'] = {
    label = 'Single Nitro',
    weight = 500,
    stack = true,
    close = true,
    description = 'Install single nitro on your vehicle',
    server = { export = 'rgarage_nitro.useNitroItem' }
},
['nitroduo'] = {
    label = 'Double Nitro',
    weight = 500,
    stack = true,
    close = true,
    description = 'Install double nitro on your vehicle',
    server = { export = 'rgarage_nitro.useNitroItem' }
},
```

That is all. The export receives ox_inventory's own signature and resolves the player
from the inventory it was used in.

> **Only want one of them?** Just add the one you need to `items.lua`. The resource
> works fine with a single item.
 Optionally drop `nitrosingle.png` / `nitroduo.png`
into `ox_inventory/web/images/`.

### Option B — qb-inventory

**1.** In `qb-core/shared/items.lua`:

```lua
nitrosingle = { name = 'nitrosingle', label = 'Single Nitro', weight = 500, type = 'item', image = 'nitrosingle.png', unique = false, useable = true, shouldClose = true, description = 'Install single nitro' },
nitroduo    = { name = 'nitroduo',    label = 'Double Nitro', weight = 500, type = 'item', image = 'nitroduo.png',    unique = false, useable = true, shouldClose = true, description = 'Install double nitro' },
```

**2.** In **any of your own server files** (never inside `rgarage_nitro`):

```lua
local QBCore = exports['qb-core']:GetCoreObject()

-- Single nitro (200)
QBCore.Functions.CreateUseableItem('nitrosingle', function(src)
    TriggerClientEvent('rgarage_nitro:client:useItem', src, 'nitrosingle')
end)

-- Double nitro (400)
QBCore.Functions.CreateUseableItem('nitroduo', function(src)
    TriggerClientEvent('rgarage_nitro:client:useItem', src, 'nitroduo')
end)
```

> **Only want one of them?** Register only the item you want. The resource works fine
> with a single item, and nothing else needs changing.


**3.** Put the images in `qb-inventory/html/images/`.

> **Do not remove the item yourself.** The resource takes it only after the install
> progress finishes, so a cancelled install costs the player nothing. Removing it in
> your own code charges them twice.

## 6. Permissions — `config.lua`

```lua
Config.Commands = {
    enabled    = true,
    nameSingle = "nitrosingle",   -- /nitrosingle
    nameDuo    = "nitroduo",      -- /nitroduo
    groups     = {}               -- empty = anyone
}
```

On QBCore the key is a **job name** or a **gang name**, and the value is the minimum
grade:

```lua
groups = {
    ["mechanic"] = 0,   -- any mechanic
    ["police"]   = 2,   -- police, grade 2 or higher
}
```

## 7. Notify and progress bar

```lua
Config.Notify   = { enabled = true, type = "ox_lib" }
Config.Progress = { enabled = true, type = "ox_lib" }
```

`ox_lib` is the right default on QB. To use qb-core's own notification instead:

```lua
Config.Notify = {
    enabled = true,
    type = "custom",
    CustomFunction = function(nType, message, duration)
        TriggerEvent('QBCore:Notify', message, nType, duration)
    end,
}
```

## 8. Nitro bottle prop (optional)

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

**Resource does not start**
`@vrp/lib/utils.lua` or `'vrp'` still active in `fxmanifest.lua`. Both must be
commented out on QBCore.

**Using the item does nothing, no error**
The server console prints `GetUserId returned nil` along with the configured
framework. Confirm `Config.Framework = "qb"` and that `qb-core` starts first.

**ox_inventory: item is consumed but nothing happens**
The `server = { export = ... }` line is missing or misspelled. It must be exactly
`rgarage_nitro.useNitroItem`.

**Command says no permission**
The key in `groups` is a job or gang name, not a QB permission level. `["admin"] = 0`
only works if `admin` is an actual job on your server.
