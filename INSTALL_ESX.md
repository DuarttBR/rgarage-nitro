# rgarage_nitro — Installation on ESX

Covers both the **ESX built-in inventory** and **ox_inventory**.

---

## 1. Dependencies

Start these **before** `rgarage_nitro` in `server.cfg`:

- `es_extended`
- `oxmysql`
- `ox_lib`

```cfg
ensure es_extended
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
Config.Framework = "esx"
Config.Language  = "en"
```

## 4. fxmanifest.lua — REQUIRED on ESX

Two lines must be **commented out**. There is no `vrp` resource on your server, so
leaving them active makes the resource fail to start:

```lua
shared_scripts {
    'config.lua',
    'Locales.lua',
    '@ox_lib/init.lua',
    -- '@vrp/lib/utils.lua'   -- <= MUST stay commented on ESX
}

dependencies {
    'ox_lib',
    'oxmysql',
    -- 'vrp'                   -- <= MUST stay commented on ESX
}
```

This is the single most common reason the resource does not boot on ESX.

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


### Option B — ESX built-in inventory

**1.** Insert the items in the database:

```sql
INSERT INTO `items` (`name`, `label`, `weight`, `rare`, `can_remove`) VALUES
    ('nitrosingle', 'Single Nitro', 1, 0, 1),
    ('nitroduo',    'Double Nitro', 1, 0, 1);
```

(Older ESX versions use a shorter `items` table — match whatever columns yours has.)

**2.** In **any of your own server files** (never inside `rgarage_nitro`):

```lua
ESX = exports['es_extended']:getSharedObject()

-- Single nitro (200)
ESX.RegisterUsableItem('nitrosingle', function(src)
    TriggerClientEvent('rgarage_nitro:client:useItem', src, 'nitrosingle')
end)

-- Double nitro (400)
ESX.RegisterUsableItem('nitroduo', function(src)
    TriggerClientEvent('rgarage_nitro:client:useItem', src, 'nitroduo')
end)
```

> **Only want one of them?** Register only the item you want. The resource works fine
> with a single item, and nothing else needs changing.


> **Do not remove the item yourself.** The resource calls `removeInventoryItem` only
> after the install progress finishes, so a cancelled install costs the player
> nothing. Removing it in your own code charges them twice.

## 6. Permissions — `config.lua`

```lua
Config.Commands = {
    enabled    = true,
    nameSingle = "nitrosingle",   -- /nitrosingle
    nameDuo    = "nitroduo",      -- /nitroduo
    groups     = {}               -- empty = anyone
}
```

On ESX each key is matched against **two** things, and either one passing is enough:

1. the permission group from `getGroup()` — `user`, `admin`, `superadmin`
2. the **job name** from `getJob().name` — `mechanic`, `police`

```lua
groups = {
    ["mechanic"] = 0,   -- job
    ["admin"]    = 0,   -- permission group
}
```

The value is ignored on ESX (it exists for QBCore grades), so `0` is fine.

## 7. Notify and progress bar

```lua
Config.Notify   = { enabled = true, type = "ox_lib" }
Config.Progress = { enabled = true, type = "ox_lib" }
```

To use ESX's own notification instead:

```lua
Config.Notify = {
    enabled = true,
    type = "custom",
    CustomFunction = function(nType, message, duration)
        TriggerEvent('esx:showNotification', message)
    end,
}
```

Note the resource's item-removal message already goes through
`esx:showNotification` from inside the bridge.

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
commented out on ESX.

**Using the item does nothing, no error**
The server console prints `GetUserId returned nil` along with the configured
framework. Confirm `Config.Framework = "esx"` and that `es_extended` starts first.

**ox_inventory: item is consumed but nothing happens**
The `server = { export = ... }` line is missing or misspelled. It must be exactly
`rgarage_nitro.useNitroItem`.

**Command says no permission**
The key has to be either the job name or the permission group, spelled exactly as
your server returns it. `getJob().name` is lowercase on most ESX setups.
