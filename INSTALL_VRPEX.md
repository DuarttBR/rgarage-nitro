# rgarage_nitro — Instalação no VRPeX

Para vRP clássico / Creative, veja `INSTALL_VRP.md`. A instalação é quase idêntica;
o que muda é o `Config.Framework` e como as permissões são verificadas.

---

## 1. Dependências

Inicie estes recursos **antes** do `rgarage_nitro` no `server.cfg`:

- `vrp` (VRPeX)
- `oxmysql`
- `ox_lib`

```cfg
ensure vrp
ensure oxmysql
ensure ox_lib
ensure rgarage_nitro
```

## 2. Banco de dados

Rode o `rgarage_nitro.sql` uma vez. Ele cria duas tabelas:

- `rgarage_nitro` — tipo, quantidade, cor e posição da garrafa, por placa
- `rgarage_nitro_player_prefs` — pressão e layout da HUD, por jogador

Colunas que estiverem faltando são criadas automaticamente ao iniciar, então quem
está atualizando de uma versão antiga não precisa fazer mais nada.

> Requer **MySQL 5.7+** ou **MariaDB 10.2+**.

## 3. config.lua

```lua
Config.Framework = "vrpex"
Config.Language  = "pt_br"
```

## 4. fxmanifest.lua — obrigatório no VRPeX

Duas linhas precisam estar **descomentadas**:

```lua
shared_scripts {
    'config.lua',
    'Locales.lua',
    '@ox_lib/init.lua',
    '@vrp/lib/utils.lua'      -- <= OBRIGATÓRIO no VRPeX
}

dependencies {
    'ox_lib',
    'oxmysql',
    'vrp'                      -- <= OBRIGATÓRIO no VRPeX
}
```

O VRPeX é acessado pelo `Proxy.getInterface("vRP")`, que só existe depois que o
`utils.lua` é carregado. Sem ele o recurso para de iniciar com o erro
*"vRP is not available"*.

> **Servidor Linux diferencia maiúscula de minúscula.** Várias builds do VRPeX vêm
> com o arquivo `lib/Utils.lua` com U maiúsculo. Abra a pasta `vrp/lib/`, confira o
> nome real e escreva exatamente igual, senão o recurso não sobe.

## 5. O item

O recurso já vem com dois itens: **`nitrosingle`** (200) e **`nitroduo`** (400),
definidos em `Config.Items`.

### 5.1 Adicione na sua itemlist

```lua
["nitrosingle"] = { "Nitro Single", "Instala nitro simples no veículo.", 0.5, ... },
["nitroduo"]    = { "Nitro Duplo",  "Instala nitro duplo no veículo.",   0.5, ... },
```

Mantenha os nomes `nitrosingle` e `nitroduo`, ou ajuste o `Config.Items` para os
nomes que você usou.

### 5.2 Deixe o item usável

Dispare o evento de client no ponto onde a sua itemlist trata o uso do item. São dois
itens, então são duas chamadas:

```lua
-- Nitro simples (200)
TriggerClientEvent('rgarage_nitro:client:useItem', source, 'nitrosingle')

-- Nitro duplo (400)
TriggerClientEvent('rgarage_nitro:client:useItem', source, 'nitroduo')
```

Se a sua build do VRPeX usa o padrão de export do inventário, registre os dois:

```lua
exports.vrpex:registerUsableItem('nitrosingle', function(source)
    TriggerClientEvent('rgarage_nitro:client:useItem', source, 'nitrosingle')
end)

exports.vrpex:registerUsableItem('nitroduo', function(source)
    TriggerClientEvent('rgarage_nitro:client:useItem', source, 'nitroduo')
end)
```

> **Quer só um dos dois?** Cadastre na itemlist apenas o item que você vai usar e
> registre só ele. O recurso funciona normalmente com um item só, e não precisa mexer
> em mais nada.

> **Não remova o item por conta própria.** O recurso só tira o item **depois** que a
> barra de instalação termina, usando o `TryGetInventoryItem` do bridge. Se você
> remover no seu código, o jogador paga duas vezes e uma instalação cancelada
> continua custando o item.

## 6. Permissões — `config.lua`

```lua
Config.Commands = {
    enabled    = true,
    nameSingle = "nitrosingle",   -- /nitrosingle
    nameDuo    = "nitroduo",      -- /nitroduo
    groups     = {}               -- vazio = qualquer um pode usar
}
```

O bridge do VRPeX é o mais flexível dos quatro: para cada chave ele tenta, nesta
ordem, `hasGroup`, depois `hasPermission`, e por último quebra nomes com ponto e
testa as duas partes. Ou seja, todas estas formas funcionam:

```lua
groups = {
    ["Mechanic"]       = 0,   -- grupo
    ["mechanic.nitro"] = 0,   -- permissão, também casa com o grupo "mechanic"
    ["Admin"]          = 0,
}
```

Use exatamente a mesma grafia (maiúsculas e minúsculas) que está na sua base.

## 7. Notificação e barra de progresso

```lua
Config.Notify   = { enabled = true, type = "ryu" }     -- "ox_lib" | "ryu" | "custom"
Config.Progress = { enabled = true, type = "event" }   -- "event" | "ox_lib" | "custom"
```

- `"ryu"` dispara `ryu_hud:notification` (Ryu-Hud)
- `"event"` dispara `TriggerEvent("Progress", duration)`, o padrão das bases vRP

Se a sua base usa o evento de notificação com cores, use `"custom"`:

```lua
Config.Notify = {
    enabled = true,
    type = "custom",
    CustomFunction = function(nType, message, duration)
        TriggerEvent("Notify", nType, message, duration)
    end,
}
```

## 8. Garrafa de nitro no veículo (opcional)

Dá para mostrar uma garrafa física no carro. O jogador posiciona ela com um editor
na hora de instalar, e a posição fica salva por placa.

```lua
Config.Prop = {
    enabled        = true,
    placeOnInstall = true,   -- posicionar a garrafa antes de começar a instalação
    withCollision  = false,  -- deixe false enquanto ela estiver presa no veículo
}
```

No editor: **botão esquerdo** arrasta a seta, **botão direito segurado** gira a
câmera, **Shift/Ctrl** sobem e descem a câmera, **PgUp/PgDn** aproximam e afastam,
**E/R/Q** alternam mover, girar e eixo local, **Enter** confirma e **Esc** cancela.

Os dois comandos auxiliares vêm **desativados**. Ative no `config.lua` só se precisar:

```lua
Config.Prop.testCommand = "nitroproptest"   -- debug: spawna o modelo e mostra o tamanho no F8
Config.Prop.editCommand = "nitrobottle"     -- permite reabrir o editor depois de instalar
```

Com `placeOnInstall = true` o jogador já posiciona a garrafa durante a instalação,
então nenhum dos dois é necessário no uso normal.

---

## Problemas comuns

**Uso o item e não acontece nada, sem erro nenhum**
O console do servidor imprime `GetUserId returned nil` junto com o framework que está
configurado. Confirme `Config.Framework = "vrpex"` e se o `@vrp/lib/utils.lua` está
descomentado.

**"vRP is not available" ao iniciar**
Falta o `utils.lua` no `shared_scripts`, o nome do arquivo está com maiúscula errada
no Linux, ou o `vrp` está iniciando depois do `rgarage_nitro`.

**A permissão sempre nega**
O bridge tenta `hasGroup` e `hasPermission`. Se as duas falham, o nome da chave não é
o que a sua base usa — confira a grafia e as maiúsculas no seu `Groups.lua`.

**O nitro não salva**
Confira a versão do MySQL (5.7+ / MariaDB 10.2+) e se o veículo tem placa. Veículo sem
placa não tem como ser diferenciado dos outros veículos sem placa, então o recurso
recusa salvar.
