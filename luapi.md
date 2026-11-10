## 2. UI & Menu System (`menu` / `ui`)

### Creating Custom Tabs & Elements
Custom tabs appear in the SynUI cheat window alongside built-in categories.

```lua
local my_tab = menu.new_tab("HVH Lua", "rbxassetid://10734975692")
local left_sec = my_tab:new_section("Resolver Settings", "Left")  -- "Left" or "Right"
```

#### Section Element Creators
| Method | Description | Return |
| :--- | :--- | :--- |
| `sec:new_toggle(name, [default], [callback])` | Checkbox toggle | `ElementHandle` |
| `sec:new_slider(name, min, max, [default], [decimals], [suffix], [callback])` | Numeric slider | `ElementHandle` |
| `sec:new_combobox(name, options, [default], [callback])` | Dropdown selector | `ElementHandle` |
| `sec:new_keybind(name, [mode], [key], [callback])` | Hotkey (`"Toggle"`, `"Hold"`, `"Always"`) | `ElementHandle` |
| `sec:new_colorpicker(name, [default], [callback])` | RGBA color picker | `ElementHandle` |
| `sec:new_textbox(name, [placeholder], [default], [callback])` | String input field | `ElementHandle` |
| `sec:new_button(name, callback)` | Clickable push button | `ElementHandle` |
| `sec:new_label(text)` | Static text label | `ElementHandle` |

#### Element Handle Methods
```lua
handle:get()                       -- Returns current element value (bool, number, string, Color3)
handle:set(value, [silent])        -- Updates element value (silent skips callbacks)
handle:set_visible(boolean)        -- Shows or hides element in the UI
handle:set_callback(function)      -- Attaches change listener
```

---

### Gamesense UI Emulation (`ui.*`)
Full drop-in compatibility with Gamesense scripts:

```lua
-- Referencing built-in cheat toggles and sliders:
local pitch_mode, pitch_slider = ui.reference("AA", "Anti-aimbot angles", "Pitch")
local body_mode, body_slider   = ui.reference("AA", "Anti-aimbot angles", "Body yaw")
local dt_toggle, dt_hotkey     = ui.reference("RAGE", "Aimbot", "Double tap")

ui.set(pitch_slider, -89)
local is_dt = ui.get(dt_toggle)
```

- `ui.new_checkbox(tab, section, name, [default]) -> ElementHandle`
- `ui.new_slider(tab, section, name, min, max, [default], [step], [unit], [scale]) -> ElementHandle`
- `ui.new_combobox(tab, section, name, options...) -> ElementHandle`
- `ui.new_multiselect(tab, section, name, options...) -> ElementHandle`
- `ui.new_color_picker(tab, section, name, r, g, b, [a]) -> ElementHandle`
- `ui.new_hotkey(tab, section, name, [inline], [key]) -> ElementHandle`
- `ui.new_button(tab, section, name, callback) -> ElementHandle`
- `ui.get(ref)` / `ui.set(ref, ...)`
- `ui.is_menu_open() -> boolean`
- `ui.menu_position() -> (number, number)`

---

## 3. Anti-Aim & Desync (`aa`)

The `aa` library provides complete authority over local camera pitch, yaw, custom mathematical modifiers, desync body angles up to **86°**, and fake lag choking.

### Pitch
- `aa.override_pitch(pitch)` / `aa.set_pitch(pitch)`: Overrides pitch angle. Accepts:
  - Numbers: `[-89..89]` degrees (e.g. `-89` down, `89` up, `0` zero).
  - Strings: `"Down"`, `"Up"`, `"Zero"`, `"Disabled"`.
  - Pass `nil` to restore menu control.
- `aa.get_pitch() -> number?`: Returns active pitch override.
- `aa.is_pitch_overridden() -> boolean`

### Yaw & Modifiers
- `aa.override_yaw(deg, [base])`: Sets base yaw in degrees `[-180..180]`.
  - `base`: `"camera"` (relative to view) or `"world"` (absolute world axis).
- `aa.override_yaw_offset(deg)`: Sets static angle offset added to base yaw.
- `aa.override_yaw_modifier(modifier, [offset])`: Custom jitter, spin, or mathematical sway:
  - **Function**: `function(tick, baseYaw) -> number (offset in degrees)`
  - **Presets**: `"Spin"`, `"Random"`, `"Center"`, `"3-Way"`, `"5-Way"`, `"Disabled"`.
- `aa.get_yaw() -> number?` / `aa.get_yaw_offset() -> number?`

### Body Yaw & Desync (Unlocked up to 86°)
- `aa.override_body_yaw(enabled)`: Enables or disables desync simulation.
- `aa.override_inverter(inverted)`: Sets desync side (`false` = Fake Left / Real Right, `true` = Fake Right / Real Left).
- `aa.override_desync_limits(leftDeg, rightDeg)`: Sets max desync rotation limits `[0..86]` degrees.
- `aa.override_desync(realDeg, fakeDeg)`: Explicitly sets Real and Fake body angles `[-86..86]`.
- `aa.override_show_fake(boolean)`: Forces thirdperson ghost visualizer on or off.
- `aa.get_inverter() -> boolean`
- `aa.get_desync_limits() -> (number, number)`: Returns `left, right` limits (up to `86, 86`).
- `aa.get_lby() -> number` / `aa.get_lby_deg() -> number`: Local Real Feet Yaw (radians/degrees).
- `aa.get_fake_lby_deg() -> number`: Local Fake Feet Yaw in degrees.
- `aa.get_twist_deg() -> number`: Delta angle between real body yaw and feet yaw.

### Fake Lag
- `aa.override_fake_lag(enabled)`: Enables/disables packet choking.
- `aa.override_fake_lag_limit(ticks)`: Sets choke limit between `0` and `14` ticks.
- `aa.override_fake_lag_variability(pct)`: Sets choke randomization percentage `[0..100%]`.
- `aa.is_fake_lag_choking() -> boolean`: `true` if current simulation tick is choked.
- `aa.unchoke()` / `aa.flush_choked()`: Instantly flushes all queued network packets.

### Reset
- `aa.reset()` / `aa.clear()`: Clears all anti-aim overrides and restores menu settings.

---

## 4. Ragebot & Resolver (`rage` / `plist`)

### Built-in Baseline Resolver Control
Control the native engine resolver directly from Lua:

| Function | Description | Values / Defaults |
| :--- | :--- | :--- |
| `rage.set_resolver_enabled(boolean)` | Enables or disables built-in baseline resolver | `true` / `false` |
| `rage.is_resolver_enabled() -> boolean` | Checks if built-in resolver is active | Returns `boolean` |
| `rage.set_resolver_mode(mode)` | Sets resolver algorithm | `"Brute-force"`, `"Adaptive"`, `"Freestanding"` |
| `rage.get_resolver_mode() -> string` | Returns active resolver mode | Returns `string` |
| `rage.set_resolver_miss_limit(n)` | Shots before angle cycles | `1` to `5` (default `2`) |
| `rage.get_resolver_miss_limit() -> number` | Returns current miss limit | Returns `number` |
| `rage.set_resolver_baim_after_misses(n)` | Auto body-aim threshold | `0` to `5` (default `2`, `0` = off) |
| `rage.get_resolver_baim_after_misses() -> number` | Returns BAim threshold | Returns `number` |
| `rage.reset_resolver_state([target])` | Resets miss counter and cycle index | Optional `target` slot |

---

### Player List Overrides (`plist`)
Provides 100% authoritative priority over any enemy's resolved yaw, hitbox angles, and targeting rules.

```lua
plist.set(target, key, value)
plist.get(target, key) -> any
plist.reset(target)
plist.reset_all()
```

#### Supported `plist` Keys:
- `"Force body yaw"` (`boolean`): Enables manual resolver angle override for this player.
- `"Force body yaw value"` / `"Override yaw"` (`number`): Sets target's feet yaw to an absolute angle in degrees `[-180..180]`.
- `"Override yaw offset"` (`number`): Sets an angle offset relative to enemy visual angle (e.g. `58`, `-58`, `0`, `86`, `-86`).
- `"Override prefer body aim"` / `"Force body aim"` (`string` | `boolean`): `"Force"` / `true` forces BAim; `"-"` / `false` restores normal scan.
- `"Override safe point"` / `"Force safe point"` (`string` | `boolean`): `"Force"` / `true` requires safe points.
- `"Force pitch"` (`boolean`) / `"Force pitch value"` (`number`): Overrides enemy pitch.

#### Quick Helper Functions:
- `rage.override_player_yaw(target, deg)`: Forces absolute yaw on target.
- `rage.override_player_yaw_offset(target, offsetDeg)`: Offsets target yaw (e.g. `58` or `-58`).
- `rage.override_player_pitch(target, deg)`: Overrides target pitch.
- `rage.reset_player(target)`: Restores target angles to default.

---

### Global Ragebot Overrides (`rage`)
- `rage.override_enabled(boolean)` / `rage.is_enabled() -> boolean`
- `rage.override_min_damage(damage)` / `rage.get_min_damage() -> number`
- `rage.override_hitchance(hitchance)` / `rage.get_hitchance() -> number`
- `rage.override_force_baim(boolean)` / `rage.is_force_baim() -> boolean`
- `rage.override_force_safe_point(boolean)` / `rage.is_force_safe_point() -> boolean`
- `rage.override_target_selection(mode)`: `"Highest damage"`, `"Near crosshair"`, `"Best hitchance"`, `"Cycle"`.
- `rage.override_hitboxes(list)`: List of parts: `{ "head", "chest", "pelvis", "arms", "legs" }`.
- `rage.override_hitbox_priority(list)`: Order of evaluation: `{ "pelvis", "chest", "head" }`.
- `rage.override_hitbox_min_damage(hitbox, dmg)`: Per-part min damage requirement.
- `rage.override_hitbox_hitchance(hitbox, hc)`: Per-part hitchance requirement.
- `rage.set_hitbox_config(hitbox, { min_damage = 80, hitchance = 65, multipoint = true })`
- `rage.get_target() -> TargetInfo?`: Active target snapshot (`slot`, `player`, `part_name`, `aim_point`, `damage`, `hitchance`, `resolved_yaw_deg`).
- `rage.get_target_info() -> PlayerInfoTable?`: Full runtime status of current target.
- `rage.reset()` / `rage.clear()`: Restores menu rage settings.

---

### Aim & Shot Event Callbacks
Listen to shots using `client.set_event_callback` or `events.on`:

```lua
-- 1. Shot fired
client.set_event_callback("aim_fire", function(e)
    -- e.id, e.target, e.hitbox, e.damage, e.hitchance, e.resolved_yaw, e.point
end)

-- 2. Authoritative server hit confirmation
client.set_event_callback("aim_hit", function(e)
    -- e.id, e.target, e.hitbox, e.damage, e.health_remaining, e.backtrack_ticks
end)

-- 3. Shot missed
client.set_event_callback("aim_miss", function(e)
    -- e.id, e.target, e.hitbox, e.reason ("resolver" | "spread" | "occlusion" | "death")
end)
```

---

## 5. Entities & Players (`entity` / `client`)

### Consolidated Player Snapshots
Instead of individual queries, use high-level snapshot functions:

```lua
-- Full player status in one zero-allocation call:
local info = entity.get_info(target)
if info and info.is_alive then
    print(info.name, info.health, info.state)  -- "Stand", "Run", "Duck", "Jumping"
    print(info.speed, info.position, info.velocity)
    print(info.yaw, info.pitch, info.lby, info.desync_delta)
    print(info.weapon_name, info.is_crouching, info.is_in_air)
end

-- Gamesense Animstate snapshot (ideal for resolvers):
local anim = entity.get_anim_state(target)
if anim then
    print(anim.eye_angles_y)                 -- Target eye yaw in degrees
    print(anim.torso_yaw)                    -- Procedural torso yaw
    print(anim.goal_feet_yaw)                -- Target LBY / Feet Yaw
    print(anim.duck_amount)                  -- 1.0 if ducking, 0.0 otherwise
    print(anim.feet_speed_forwards_or_sideways)
    print(anim.desync)                       -- Evaluated desync delta in degrees
end
```

### Enumeration & Queries
- `entity.get_enemies() -> { Player }`
- `entity.get_teammates() -> { Player }`
- `entity.get_players([enemiesOnly]) -> { Player }`
- `entity.get_local_player() -> Player`
- `entity.is_alive(target) -> boolean`
- `entity.is_enemy(target) -> boolean`
- `entity.get_player_health(target) -> number`
- `entity.get_player_armor(target) -> number`
- `entity.get_player_position(target) -> Vector3`
- `entity.get_player_velocity(target) -> Vector3`
- `entity.get_player_speed(target) -> number`
- `entity.get_player_angles_deg(target) -> (pitch, yaw)`
- `entity.get_player_hitbox_position(target, hitbox) -> Vector3?`: Hitboxes: `"head"`, `"chest"`, `"pelvis"`, `"right_arm"`, `"left_arm"`, `"right_leg"`, `"left_leg"`.
- `entity.get_player_weapon_name(target) -> string`
- `entity.is_player_scoped(target) -> boolean`
- `entity.get_player_lby_deg(target) -> number`
- `entity.get_player_twist_deg(target) -> number`

### Local Weapon & Camera (`client`)
```lua
local wep = client.get_weapon()
print(wep.id, wep.name, wep.type)      -- e.g. "Scout", "SSG 08", "SniperRifle"
print(wep.ammo, wep.reserve)            -- Clip and reserve ammo
print(wep.scope_level, wep.is_scoped)   -- Scope magnification level (0, 1, 2)

local pitch, yaw = client.get_pitch_yaw()
local pDeg, yDeg, roll = client.get_camera_angles_deg()
local pingSec = client.latency()
```

---

## 6. 2D Canvas & Drawing (`render` / `Drawing`)

Hardware-accelerated, resolution-independent drawing system with zero-allocation object pooling.

### Helper Constructors
```lua
local v2 = vector(100, 200)         -- Vector2
local v3 = vector(10, 20, 30)       -- Vector3
local c1 = color(255, 60, 70)       -- Color3 from RGB
local c2 = color("#00FFA0")         -- Color3 from HEX
```

### Direct Render Functions (`render.*` / `renderer.*`)
- `render.text(x, y, text, [color], [size], [outline], [center], [font], [zIndex]) -> TextDrawing`
- `render.rect(x, y, w, h, [color], [rounding], [thickness], [zIndex]) -> SquareDrawing`
- `render.rect_filled(x, y, w, h, [color], [rounding], [zIndex]) -> SquareDrawing`
- `render.gradient(x, y, w, h, col1, col2, [vertical], [rounding], [zIndex]) -> SquareDrawing`
- `render.line(x1, y1, x2, y2, [color], [thickness], [zIndex]) -> LineDrawing`
- `render.circle(x, y, radius, [color], [thickness], [zIndex]) -> CircleDrawing`
- `render.circle_filled(x, y, radius, [color], [zIndex]) -> CircleDrawing`
- `render.triangle(p1, p2, p3, [color], [filled], [thickness], [zIndex])`
- `render.image(x, y, w, h, assetId, [color], [rounding], [zIndex])`
- `render.indicator(r, g, b, a, text...) -> number`: Draws a stacked HUD indicator in the bottom-left corner.
- `render.measure_text(text, [size], [font]) -> Vector2`: Measures rendered pixel size.
- `render.screen_size() -> (w, h)`: Viewport dimensions in pixels.
- `render.world_to_screen(worldPos: Vector3) -> (x, y, onScreen)`: 3D to 2D projection.
- `render.clear()`: Recycles all drawings back to pool.

### Standard `Drawing.new` Interface
Supported primitives: `"Text"`, `"Square"`, `"Line"`, `"Circle"`, `"Triangle"`, `"Quad"`, `"Image"`.
Fonts: `Drawing.Fonts.UI` (0), `System` (1), `Plex` (2), `Monospace` (3).

```lua
local txt = Drawing.new("Text")
txt.Text = "Custom Watermark"
txt.Position = Vector2.new(20, 20)
txt.Color = Color3.fromRGB(255, 255, 255)
txt.Size = 13
txt.Outline = true
txt.Visible = true

txt:Remove() -- Recycles object
```

---

## 7. Events & Simulation Hooks (`events` / `client`)

### 64 Hz Client Simulation Ticks
Callbacks run synchronously inside the movement and usercmd prediction loop:

```lua
-- Runs every simulation tick (64 Hz / ~15.6ms)
local handle = events.on_tick(function(tick)
    -- Simulation code
end)

-- Runs every N ticks (e.g. 4 ticks = 16 Hz)
events.every_ticks(4, function(tick) end)

-- Single-shot delayed execution
events.delay_ticks(10, function(tick) end)

-- Cancel a listener
handle:remove() -- or events.remove_listener(handle)
```

### Frame & Render Callbacks
- `client.set_event_callback("paint", function() ... end)`: Render hook (draw canvas overlays).
- `client.set_event_callback("tick", function(tick, dt) ... end)`: Simulation tick hook.
- `client.set_event_callback("shutdown", function() ... end)`: Executed on script unload (clean up memory).

---

## 8. Input Polling (`input`)

- `input.is_key_down(key)`: Returns `true` if keyboard key (e.g. `"W"`, `"E"`, `"LeftShift"`) is held.
- `input.is_mouse_down(btn)`: Returns `true` if mouse button (`1` = Left, `2` = Right, `3` = Middle) is held.
- `input.get_mouse_position() -> (x, y)`: Viewport cursor coordinates in pixels.
- `input.is_typing() -> boolean`: `true` if typing in Roblox chat or a focused UI textbox.
- `input.on_key_press(key, callback)`: Triggers on key press.

---

## 9. Utilities & Globals

### Time & Engine Counters (`globals`)
- `globals.curtime() -> number`: Server-synchronized game time in seconds.
- `globals.tickcount() -> number`: Authoritative simulation tick count.
- `globals.tickinterval() -> number`: Duration of one tick (`0.015625s`).
- `globals.realtime() -> number`: Local system clock (`os.clock()`).
- `globals.frametime() -> number`: Delta time of the last render frame.

### In-Memory Database (`database`)
Persists runtime data and settings across script reload sessions:
```lua
database.write("my_cfg", { angle = 58, mode = "adaptive" })
local cfg = database.read("my_cfg")
database.flush()
```

### Bitwise Operations (`bit` / `bit32`)
Optimized 64-bit integer bitwise operations:
`bit.band`, `bit.bor`, `bit.bxor`, `bit.bnot`, `bit.lshift`, `bit.rshift`.

### JSON Serialization (`json`)
- `json.stringify(table) -> string`
- `json.parse(jsonString) -> table`

### Console Variables (`cvar`)
- `cvar.<name>:get_float()`, `:get_int()`, `:get_string()`
- `cvar.<name>:set_float(val)`, `:set_int(val)`, `:set_string(val)`

---

## 10. Example Scripts

### Example 1: Brute-Force HvH Resolver with Dynamic BAim & HUD

```lua

local my_tab = menu.new_tab("Resolver", "rbxassetid://10734975692")
local sec = my_tab:new_section("Resolver Settings", "Left")
local hud_sec = my_tab:new_section("Display", "Right")

local enabled = sec:new_toggle("Enable Lua Resolver", true)
local baim_miss = sec:new_slider("BAim After Misses", 1, 5, 2, 0, " misses")
local show_hud = hud_sec:new_toggle("Show Debug HUD", true)

-- Per-target resolver state
local targetState = {}
local function getState(slot)
    if not targetState[slot] then
        targetState[slot] = { misses = 0, stage = 1 }
    end
    return targetState[slot]
end

-- Brute-force angle sequence relative to visual yaw
local BRUTE_ANGLES = { 58, -58, 0, 86, -86 }

-- Shot Event Hooks
client.set_event_callback("aim_hit", function(e)
    local state = getState(e.target)
    state.misses = 0 -- Reset misses on hit
end)

client.set_event_callback("aim_miss", function(e)
    local state = getState(e.target)
    if e.reason == "resolver" then
        state.misses = state.misses + 1
        state.stage = (state.stage % #BRUTE_ANGLES) + 1
    end
end)

-- 64 Hz Resolver Loop
events.on_tick(function(tick)
    if not enabled:get() then
        plist.reset_all()
        return
    end

    local enemies = entity.get_enemies()
    for _, enemy in ipairs(enemies) do
        if entity.is_alive(enemy) then
            local slot = entity.get_player_slot(enemy) or enemy
            local state = getState(slot)

            -- Apply resolved angle offset
            local offset = BRUTE_ANGLES[state.stage] or 0
            plist.set(enemy, "Force body yaw", true)
            plist.set(enemy, "Override yaw offset", offset)

            -- Force BAim when threshold reached
            if state.misses >= baim_miss:get() then
                plist.set(enemy, "Override prefer body aim", "Force")
            else
                plist.set(enemy, "Override prefer body aim", "-")
            end
        end
    end
end)

-- On-Screen Watermark & Indicator
client.set_event_callback("paint", function()
    if not show_hud:get() or not enabled:get() then return end

    local cur = rage.get_target()
    if cur and cur.slot ~= nil then
        local state = getState(cur.slot)
        local ang = BRUTE_ANGLES[state.stage] or 0
        render.indicator(0, 255, 160, 255, string.format("RESOLVER: %d° | MISS: %d", ang, state.misses))
    else
        render.indicator(200, 200, 200, 255, "RESOLVER: ACTIVE")
    end
end)

-- Cleanup on unload
client.set_event_callback("shutdown", function()
    plist.reset_all()
    render.clear()
end)
```

---

### Example 2: Custom Anti-Aim Controller (86° Desync & Fake Lag)

```lua
local my_tab = menu.new_tab("Anti-Aim", "rbxassetid://10734975692")
local sec = my_tab:new_section("AA Settings", "Left")

local aa_toggle = sec:new_toggle("Enable Custom AA", true)
local pitch_sel = sec:new_combobox("Pitch", { "Down", "Up", "Zero" }, "Down")
local desync_lim = sec:new_slider("Desync Limit", 0, 86, 86, 0, "°")
local lag_ticks = sec:new_slider("Fake Lag", 0, 14, 12, 0, "t")
local invert_key = sec:new_keybind("Invert Side", "Toggle", "F")

events.on_tick(function(tick)
    if not aa_toggle:get() then
        aa.reset()
        return
    end

    -- Pitch & Backward Yaw
    aa.override_pitch(pitch_sel:get())
    aa.override_yaw(180, "camera")

    -- Mathematical Sway Jitter Modifier
    aa.override_yaw_modifier(function(t, baseYaw)
        return math.sin(t * 0.2) * 35
    end)

    -- Desync Unlocked up to 86°
    local limit = desync_lim:get()
    aa.override_body_yaw(true)
    aa.override_inverter(invert_key:get())
    aa.override_desync_limits(limit, limit)

    -- Fake Lag Choking
    aa.override_fake_lag(true)
    aa.override_fake_lag_limit(lag_ticks:get())
end)

client.set_event_callback("paint", function()
    if not aa_toggle:get() then return end
    local choking = aa.is_fake_lag_choking()
    local side = aa.get_inverter() and "RIGHT" or "LEFT"
    render.indicator(240, 60, 70, 255, string.format("AA: 86° [%s] | CHOKE: %s", side, choking and "YES" or "NO"))
end)

client.set_event_callback("shutdown", function()
    aa.reset()
    render.clear()
end)
```
