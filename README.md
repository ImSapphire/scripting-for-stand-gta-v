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

As of natives library version `2944b`, the following flavours are available:

| Flavour | Description |
| ------- | ----------- |
|    g    | Omits namespaces, for example `PLAYER.PLAYER_ID` would become just `PLAYER_ID`.

A list of all scripting commands is available [here](https://nativedb.dotindustries.dev). Keep in mind type names displayed will be C-style, so `const char*` == `string`.

An example of using scripting commands:
```lua
util.require_natives("2944b")

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
