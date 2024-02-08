+++
title = 'Scrinks!'
date = 2024-02-08T15:50:41Z
tags = ['gamedev', 'c++', 'lua']
series = ['scrinks-dev']
+++

# Scrinks?

[Scrinks](https://github.com/blam23/ScrinksEngine) is a *(toy, very wip)* game engine written in C++ and Lua. It's not meant to be anything more than a testbed for some of my crazier ideas, as well as a project to help myself learn more lower-level OpenGL, and maybe Vulkan in the future.

I'm not very good at coming up with names, sorry! :)

# A quick tour of a new and basic project

Note that all of this is subject to change, so think of this as a snapshot of how to setup a project at the time of writing.

### Project files

First you'll want to setup your `project.lua`:

```lua
return {
    name = "Space Demo",
    entry = "scripts/game.lua",
    render = "2d",

    classes = [
        "scripts/bullet.lua",
        "scripts/player.lua",
    ]
}
```

Well, there's a few things here to point out:
 - This config is entirely lua, so it can execute code!
 - We define an entrypoint lua script, which requires an `init` function defined which is called once the setup is complete and the game loop is ready.
 - The list of classes is hardcoded (but as this is just lua we could find files on disk)
 - The engine currently has two pipelines: a WIP 3D deferred renderer `3d`, and a very basic 2D sprite renderer `2d` which we're using here.

### Initialisation

From there we need to instantiate some data once our game loop is about to run, from the `scripts/game.lua` mentioned above:
```lua
function init()
    layer = sprite_layer.new()
        :sheet("textures/space.png")
        :tiles_per_row(16)
        :tile_size(vec2.new(64,64))
        :instance_count(100000)
        :init()

    local ship = player.new()
        :position(100, 100)
        :layer(layer)
end
```

Here we see the `init` function which sets up some basic nodes for our game. Note that while the `ship` variable is a local in lua, it's memory is managed by the node system, as we don't specify a parent in `new`, it defaults to being owned by the root node (which is the node that owns this entry lua script).

A few notable things here:
- This is not very lua-ish, the syntax will likely be updated to pass in a table of parameters instead of the current method chaining approach, but this requires a bit of thinking to get right when including properties, which will be discussed below.
- `sprite_layer` is a built-in node that provides efficient instanced-based sprite rendering, although currently the only per-instance data is a position and tile index. This will eventually be updated to support scaling, rotation, flipping, & tinting as requested (so that you don't need to have rotation if nothing is rotated such as in a tile-map).
- `player` is a lua-defined class (from `scripts/player.lua` in the classes array we defined earlier). We can see below how it's setup to have a base class of the `Sprite` node.

### Defining a Lua class

`player` class: `scripts/player.lua`:
```lua
require "lib.key_map"
require_asset "space.timer"

__base_node = sprite
__props = {
    move_speed = 7
}

-- move stuff
local move_target_x = 0
local y_height = 100

-- etc...
```

This script will be loaded before initialisation, as it was defined in the `classes` array in the project. During this pre-processing the file will be interpreted and then certain values will be picked out to help create the `player` class. The `__base_node` object is a type of built in node that serves as the base-node that is created whenever a `player` object is new'd, this is somewhat similar to Godot where scripts can extend built-in classes.

We can also see the `__props` table, this will setup property data that is stored on the node (outside of lua). This is important: Any lua variables declared in the script will persist in the lua state for that node, but can't be accessed outside of that lua state. While it would be possible to access this data from other objects, it creates issues with threading (as lua states are inherently thread unsafe). Property data is therefore stored in the Node and specifically defined here, although you can add properties dynamically if required, this syntax just sets properties with whatever default values you want. So for example `move_target_x` won't be accessible from any other node or script, but `move_speed` is accessible both in the script, from `self:move_speed` as well as settable/gettable directly from other scripts, for example:

```lua
local bullet = bullet.new()
    :tile_index(17)
    :position(x + 16 + vec.x * 20, y + vec.y * 20)
    :layer(self:layer())

bullet.velocity = vec
bullet.hit_player = false
bullet.hit_enemy = true
```

Here we are setting the *properties* of `velocity`, `hit_player` and `hit_enemy`, then later these are accessed by the bullet.lua:

```lua
__base_node = sprite
__props = {
    velocity = vec2.new(0,-1),
    move_speed = 30,
    hit_player = false,
    hit_enemy = false,
}

function fixed_update()
    self:translate(self.velocity * self.move_speed)

    -- etc...
end
```

Here we can see the properies being accessed inside `fixed_update` using the `self` directive. In the previous code block, the `move_speed` property is never set which means it will be it's default value of `30`.

You may have noticed `self` throughout these code examples, I'm not sure if it's the best to overload this value as it has specific meaning in lua (e.g. I could use `this` or something else) but it creates a nice consistent syntax.

### Require

One thing to touch on briefly, going back to the first class example, is `require` and `require_asset`:
```lua
require "lib.key_map"
require_asset "timer"
```

Here we see two require methods: One imports Scrinks' built in `key_map` library, the other imports a custom user-defined `scripts/timer.lua` file, which is a more typical lua class that exists outside of the Node ecosystem and will be memory-managed by lua (although the lua state's lifetime is managed by the node that owns the script).

```lua
require 'lib.class'

timer = {}
local timer_mt = class(timer)

function timer:new(cooldown)
    return setmetatable({ cooldown = cooldown, current = 0, tick_rate = 1.0 }, timer_mt)
end

function timer:tick()
    -- note: fixed_delta is a builtin that the engine defines.
    self.current = self.current - (fixed_delta * self.tick_rate)

    if self.current <= 0 then
        self.current = self.cooldown
        return true
    end

    return false
end
```

