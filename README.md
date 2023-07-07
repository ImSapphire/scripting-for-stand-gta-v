# An introduction to making Lua/Pluto scripts for Stand for GTA V

If you've never used Lua before, read [Programming in Lua](https://www.lua.org/pil/contents.html) and the [Reference Manual](https://www.lua.org/manual/5.4/) before continuing.
Pluto is a relatively recent addition to Stand. For documentation on its features, read the [Pluto Documentation](https://pluto-lang.org/docs/Introduction).

This document does not cover all bases, it is meant to teach beginners about concepts related to RAGE (GTA V's engine), GTA V itself, and Stand. For full documentation on Stand's API functions, view the [Lua API Documentation](https://stand.gg/help/lua-api-documentation).

## Table Of Contents

* [Tooling](#tooling)
* [Getting started](#getting-started)
* [Adding commands/options to the menu](#adding-commandsoptions-to-the-menu)
* [Interacting with the game](#interacting-with-the-game)
* [Script globals and locals](#script-globals-and-locals)
* [Practical examples](#practical-examples)

## Tooling

I strongly recommend usage of an IDE for programming.

An IDE (Integrated Development Environment) is a program for developers with extra tools built-in to make creating software easier.
For scripting, you only really need a lightweight text editor, for this I recommend [Sublime Text](https://www.sublimetext.com/), however another popular choice is [Visual Studio Code](https://code.visualstudio.com/). Additionally, plugins like [Pluto Syntax Highlighting](https://github.com/PlutoLang/Syntax-Highlighting) and [Pluto Language Server](https://github.com/PlutoLang/pluto-language-server) may aid you in finding bugs while programming.

## Getting started

To get started with making a script, create a new file in `%appdata%\Stand\Lua Scripts` with the `.lua` or `.pluto` extension. Keep in mind you will need to install [Pluto Syntax Highlighting](https://github.com/PlutoLang/Syntax-Highlighting) if you use Pluto features, or if you use the `.pluto` file extension.

## Adding commands/options to the menu

All command types are listed on the [Lua API Documentation](https://stand.gg/help/lua-api-documentation#menu-functions).
You can add commands almost anywhere in the menu, however for this example we will be adding it to your script's root/list.

First, get a reference to the list you want to add a command to.
Then, you can use functions like `menu.action`, `menu.toggle`, and other functions listed in the documentation. You may also use the shorthand syntax.

OG syntax for adding commands:
```lua
local root = menu.my_root()

menu.action(root, "Your Command Name Here", {"yourcmdname1", "yourcmdname2"}, "Command Description", function(click_type, effective_issuer)
	util.toast(players.get_name(effective_issuer) .. " used the action!")
end)
```
Shorthand syntax:
```lua
local root = menu.my_root()

root:action("Your Command Name Here", {"yourcmdname1", "yourcmdname2"}, "Command Description", function(click_type, effective_issuer)
	util.toast(players.get_name(effective_issuer) .. " used the action!")
end)
```

If you do not want the command to have any command names, simply provide an empty table (`{}`) for command_names.

## Interacting with the game

To do things like spawning and modifying entities, it is recommended to invoke the game's scripting commands (commonly referred to as natives). These are intended for the game's own scripts, which handle most gameplay elements.

You will need to call `util.require_natives` to easily invoke these commands. For development purposes, calling this with no arguments (or version being nil) is fine, however in a release script you should specify the natives lib version you used at the time of development. You can find all library versions in the [Repository](https://stand.gg/focus#Stand%3ELua%20Scripts%3ERepository), all of which are prefixed with `natives-`.

As of natives library version `2944a`, the following flavours are available:

| Flavour | Description |
| ------- | ----------- |
|    g    | Omits namespaces, for example `PLAYER.PLAYER_ID` would become just `PLAYER_ID`.
|   uno   | Not type safe, so for example integers will not be automatically converted to floats. Vector3 instances will be pushed as 3 floats instead of a reference/pointer, unless you use `memory.addrof`. This means you can substitute `pos.x, pos.y, pos.z` with `pos`. |
|  g-uno  | A combination of the above two flavours. |

A list of all scripting commands is available [here](https://nativedb.dotindustries.dev). Keep in mind type names displayed will be C-style, so `const char*` == `string`.

An example of using scripting commands:
```lua
util.require_natives("2944a")

-- Teleport forward
-- Using players.user_ped() here is faster, but for the example this is fine.
local pos = ENTITY.GET_OFFSET_FROM_ENTITY_IN_WORLD_COORDS(PLAYER.PLAYER_PED_ID(), 0.0, 5.0, 0.0)
ENTITY.SET_ENTITY_COORDS(PLAYER.PLAYER_PED_ID(), pos.x, pos.y, pos.z, false, false, false, false)
```

## Script globals and locals

These are similar to Lua globals and locals, however in Rockstar's scripting language the script VM uses an integer to denote which global or local to access. You can find them by looking in the [decompiled scripts](https://github.com/Primexz/GTAV-Decompiled-Scripts).

Converting a script global or local to something usable in your script, do the following:

* Replace `.f_` with ` + `
* Replace `[` with ` + 1 + `
	* Script arrays have a hidden integer at the beginning that stores the size of the array. The purpose of the `+ 1` is to skip over that.
	* If the array is an array of script structs, it will have the member struct size as a comment. For example, `Global_2657704[PLAYER::PLAYER_ID() /*463*/]`. In this case, you would convert that to `2657704 + 1 + (PLAYER.PLAYER_ID() * 463)`
* Replace `]` with nothing

Some examples:

* `Global_2657704[PLAYER::PLAYER_ID() /*463*/].f_321.f_11` => `2657704 + 1 + (PLAYER.PLAYER_ID() * 463) + 321 + 11`
* `Global_2652364.f_2650` => `2652364 + 2650`
* `Global_100885.f_205[58]` => `100885 + 205 + 1 + 58`

To access script globals or locals, you need to get its address using `memory.script_global`/`memory.script_local` then access it using the `memory.read_*` or `memory.write_*` functions (refer to [the API documentation](https://stand.gg/help/lua-api-documentation#memory-functions)). Example:
```lua
-- Read from script global Global_2657704[PLAYER::PLAYER_ID() /*463*/].f_245
-- In this example, the global below is an int, since it's set to the return value of GET_INTERIOR_FROM_ENTITY. You can replace PLAYER.PLAYER_ID with someone else's player ID to get their interior ID.
memory.read_int(memory.script_global(2657704 + 1 + (PLAYER.PLAYER_ID() * 463) + 245))

-- Writing to globals is similar, just replace read_ with write_ and use whatever value you want to write as the second argument:

-- Global_1575043[Global_1574918] = 1
-- Enters spectator mode.
memory.write_int(memory.script_global(1575043 + 1 + memory.read_int(memory.script_global(1574918))), 1)
```

## Practical examples

To use models, in most cases you need to request them first. Stand's API has a function for requesting models, `util.request_model`. Example of using this to spawn a vehicle:
```lua
util.require_natives("2944a")

local function spawnVehicle(model_name, pos, heading = 0.0)
	local hash = util.joaat(model_name)
	util.request_model(hash)
	local vehicle = entities.create_vehicle(hash, pos, heading) -- Stand expects a Vector3, so we don't need to do pos.x, pos.y, pos.z
	STREAMING.SET_MODEL_AS_NO_LONGER_NEEDED(hash) -- Allow the engine to remove the model from memory when it is no longer being used
	return vehicle
end

local veh = spawnVehicle("adder", ENTITY.GET_ENTITY_COORDS(PLAYER.PLAYER_PED_ID()))
ENTITY.SET_ENTITY_INVINCIBLE(veh, true) -- Make the vehicle godmode
PED.SET_PED_INTO_VEHICLE(PLAYER.PLAYER_PED_ID(), veh, -1) -- Put the user into the vehicle's driver seat (seat indices start at -1)
```

Keep in mind model names are not the same as they display in game, e.g. the Space Docker's model name is actually `dune2`.

Stand also offers the functionality of getting all of a type of entity in your loaded session. This can be used for all sorts of things, one of which can be to teleport all pickups to yourself;

```lua
util.require_natives("2944a")

local function teleportPickups(v3Coords)
    local all_pickups = entities.get_all_pickups_by_handle()

    for _, pickup in pairs(all_pickups) do --iterate over the table in pairs, with "pickup" being every pickup in the table.
        NETWORK.NETWORK_REQUEST_CONTROL_OF_ENTITY(pickup) --have to gain control to modify entity
        ENTITY.SET_ENTITY_COORDS(pickup, v3Coords.x, v3Coords.y, v3Coords.z, false, false, false, false) --set the coords
    end
end
```
In this function, notice how we have to get control? This is because before you can modify entities in GTA, you need to take control of them. This is a simple function that doesn't have control checking, so it will probably skip over pickups that we don't have control over. This can be circumvented for future cases with a function;

```lua
--assume we already required natives somewhere above in the script
local function requestControl(entity, iterations)
    local i = 0
    while (not (NETWORK.NETWORK_HAS_CONTROL_OF_ENTITY(entity))) and (i < iterations) do
        NETWORK.NETWORK_REQUEST_CONTROL_OF_ENTITY(entity)
        util.yield()
    end

    if NETWORK.NETWORK_HAS_CONTROL_OF_ENTITY(entity) then
        return true
    else
        return false
    end
end
```
This simple function requests control of an entity every millisecond (hence the `util.yield()`) util either the `iterations` have gone over the specified argument, or the entity has been taken control of. We return with a `boolean` of whether or not the entity has been taken control over, and can use that to determine if we should even try to modify the entity in any awy; since if we don't have control, we can't!

## Resources (Reference Section)
This will contain the resource list that might aid you on your scripting journey.

| Resource | Details |
| ---- | ---- |
| [Native Documentation](https://nativedb.dotindustries.dev/natives) | Contains the native functions from GTA V. These are the functions that govern the game, and we can use them via the menu. Almost every single basic feature can be achieved with a combination of these. |
| [Stand LUA Documentation](https://stand.gg/help/lua-api-documentation) | Contains the functions necessary to add features to Stand. Everything from buttons to sliders to submenus, and even simpler versions of native functions can be found here. |
| [Pluto Documentation](https://pluto-lang.org/docs/Introduction) | Documentation for the `pluto` language that Stand now uses, as it offers more functionality than LUA, while still maintaining backwards-compatibility. Made by well-known member and staff *well in that case*. |
| [Data Dumps](https://github.com/DurtyFree/gta-v-data-dumps) | This is a repository of well-known data dumper durtyfree on GitHub, and is extremely helpful for developing scripts. |
| [Hash Database](https://github.com/calamity-inc/gta-v-joaat-hash-db) | An alternative (and possibly more comprehensible) version of the Data Dumps, which contains JOAAT (Jenkins-One-At-A-Time) hashes for GTA V, including pickups, weapons, and more. |
| [Decompiled Scripts](https://github.com/Primexz/GTAV-Decompiled-Scripts) | These are scripts that govern GTA V, which have been decompiled for your convenience. Global variables and the way missions work are detailed in here, if you can read them. Keep in mind that this includes not just GTA Online scripts, but GTA V ones as well, meaning if you don't look where you need to, you'll be led on a wild goose chase. |
| [UnknownCheats GTAV Forum](https://www.unknowncheats.me/forum/grand-theft-auto-v/) | This houses more experienced developers that develop cheats and try to look for exploits in GTA V. |
| [Yimura's Offsets](https://github.com/Yimura/GTAV-Classes) | In making his open-source GTA V menu YimMenu, Yimura has provided us with the offsets, which can be useful if you are screwing around with memory and offsets in your scripts. |
| And, of course, the community! | Have a look around other scripts on GitHub or the Repository, but remember; ***copy-pasting is not the way to learn, it's the way to look like a fool***. And don't be afraid to ask questions in the Discord `#programming` channel! |
