## 1. Luau Syntax, Typing & Asynchronous Runtime

Lua 5.1 type.

### 1.1 Type Annotations & Declarations
Luau type annotations are parsed natively and skipped at runtime with zero overhead, allowing scripts written with full type checking to run smoothly.

- **Function Parameters & Return Types**:
```luau
local function calculate_angle(base: number, offset: number?): (number, boolean)
    local result: number = base + (offset or 0)
    return result, true
end
```
- **Local Variable Annotations**:
```luau
local is_active: boolean = true
local target_name: string? = nil
local player_list: { [number]: string } = {}
```
- **Type Assertions (`::`)**:
```luau
local entity = get_target() :: any
local cast_num = raw_val :: number
```
- **Type Aliases (`type` / `export type`)**:
```luau
type ResolverRecord = {
    yaw: number,
    pitch: number,
    lby: number,
    choked: number,
}
export type AngleModifier = (number, number) -> number
```
- **Generics**:
```luau
local function get_or_default<T>(val: T?, fallback: T): T
    return if val ~= nil then val else fallback
end
```

### 1.2 Compound Assignment Operators
All standard Luau compound assignments are supported and desugared cleanly:
```luau
counter += 1      -- desugars to counter = counter + 1
health -= damage  -- desugars to health = health - damage
multiplier *= 1.5 -- desugars to multiplier = multiplier * 1.5
divider /= 2      -- desugars to divider = divider / 2
modVal %= 360     -- desugars to modVal = modVal % 360
powVal ^= 2       -- desugars to powVal = powVal ^ 2
str ..= "!"       -- desugars to str = str .. "!"
```

### 1.3 Flow Control: `continue`
The `continue` keyword is supported in all loop structures (`for`, `while`, `repeat until`):
```luau
for i = 1, 10 do
    if i % 2 == 0 then
        continue -- Skips even numbers, jumps to next iteration
    end
    print("Odd:", i)
end
```

### 1.4 Loop Budget Protection & Zero-Lag Architecture
A frequent issue in user scripts is heavy `while true do` loops causing severe frame drops or game freezes.
The LuaEngine includes an **automatic cooperative time-slicing loop budget**:
- **5ms Auto-Yield Budget**: Every 128 loop iterations, the interpreter inspects elapsed CPU time. If more than 5ms elapsed in the current frame, it automatically yields via `task.wait()`, returning control to Roblox's render pipeline and keeping game FPS at 60+.
- **Script Lifecycle Auto-Termination**: When a script is unloaded or reloaded via the menu, active background loops instantly terminate rather than lingering as orphan threads.

### 1.5 Asynchronous Task & Timer Library (`task`, `wait`, `delay`, `spawn`)
Full access to Roblox's modern `task` scheduler and legacy coroutine timers:

```luau
-- Yielding:
task.wait(0.5)               -- Yields execution for 0.5 seconds
wait(0.1)                    -- Alias for task.wait

-- Background Spawning:
task.spawn(function()
    while true do
        task.wait(1)
        print("Heartbeat tick:", tick())
    end
end)
spawn(function() ... end)    -- Alias for task.spawn

-- Delayed Execution:
local timer = task.delay(3, function()
    print("3 seconds elapsed!")
end)
delay(3, function() ... end) -- Alias for task.delay

-- Deferred & Cancellation:
task.defer(function() ... end)
task.cancel(timer)

-- Timing Primitives:
local now = tick()           -- Cheat engine synchronized tick time
local runTime = time()       -- Game session time
```

---

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

### Fake Lag & Choke Manipulation
- `aa.override_fake_lag(enabled)`: Enables/disables packet choking.
- `aa.override_fake_lag_limit(ticks)`: Sets choke limit between `0` and `14` ticks.
- `aa.override_fake_lag_variability(pct)`: Sets choke randomization percentage `[0..100%]`.
- `aa.is_fake_lag_choking() -> boolean`: `true` if current simulation tick is choked.
- `aa.get_choked_ticks() -> number`: Current number of choked ticks in active packet sequence.
- `aa.get_choke_target() -> number`: Target tick count for current choke sequence.
- `aa.unchoke()` / `aa.flush_choked()`: Instantly flushes all queued network packets to create teleport/burst effect.
- `aa.force_teleport()`: Flushes queued packets to instantly shift simulation origin and break lag compensation.

### Building Custom Defensive Anti-Aim in Lua
Instead of rigid hardcoded engine options, developers can build completely custom defensive anti-aims by combining low-level primitives:
- `rage.shift_ticks([ticks])` / `rage.force_shift([ticks])`: Shifts simulation ticks (DoubleTap tickbase shift) to break simulation records.
- `rage.is_double_tap_charged() -> boolean`: Returns whether DT tickbase is fully charged (`>= 14` ticks).
- `rage.get_double_tap_charge() -> number`: Returns charged tick count `[0..14]`.
- `aa.override_pitch(pitch)`: Rapidly flicks pitch (`-89`, `89`, `"Random"`).
- `aa.override_yaw_offset(deg)` / `aa.override_yaw_modifier(fn)`: Rapidly rotates, jitters or spins head angles during choked bursts or on keybind.
- `aa.unchoke()`: Releases burst unchoke when desired.
- When an enemy aimbot/resolver observes a rapid angle flick (`> 120°` in 1 frame) or broken simulation ticks, their resolver gets thrown off (`defensive_broken`), missing your head!

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

Hardware-accelerated, resolution-independent screen-space drawing system with zero-allocation object pooling, hierarchical parenting, and built-in cursor hit-testing. Designed for creating completely custom, standalone graphical interfaces (GUIs), HUDs, and overlays.

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

---

### Custom GUI & Drawing Objects (`Drawing.new`)
Supported primitives: `"Text"`, `"Square"`, `"Line"`, `"Circle"`, `"Triangle"`, `"Quad"`, `"Image"`.  
Fonts: `Drawing.Fonts.UI` (0), `System` (1), `Plex` (2), `Monospace` (3).

#### Advanced Object Properties & Hierarchical Parenting
All Drawing objects now support hierarchical parenting, allowing you to build fully nested custom GUIs (windows, panels, tabs, buttons, checkboxes, sliders) where children move and clip relative to their parent container:

| Property | Primitives | Description |
| :--- | :--- | :--- |
| `Parent` | All Primitives | Sets parent Drawing object or Roblox GuiObject. Child coordinates become relative to parent! |
| `Rounding` | `Square`, `Image` | Corner rounding radius in pixels via GPU hardware (`UICorner`). |
| `BorderColor` | `Square` | Independent stroke/outline color for filled rectangles. |
| `ClipsDescendants`| `Square` | Enables clipping so child elements inside scrollable/nested panels don't overflow. |
| `Gradient` | `Square` | Table `{ Color1, Color2, [Angle], [Transparency1], [Transparency2] }` for smooth color blends. |
| `ImageColor` | `Image` | Tint/multiply color for loaded image textures. |

#### Object Hit-Testing, Touch & Interactive Methods
Drawing objects (`Square`, `Circle`, `Text`, `Image`) provide a full suite of interactive methods with hardware-accurate hit-testing, single-click edge-detection, mobile touch support, and draggable window bindings:

```lua
-- Single-Click Edge Detection (triggers once per press, does NOT cycle when held down!)
local is_clk = my_btn:IsClicked([btn])        -- Alias for IsJustPressed; ideal for buttons and toggles
local is_prs = my_btn:IsJustPressed([btn])    -- Returns true only on the initial frame mouse/touch went down

-- Continuous Hold & Polling (for sliders, repeaters, or custom drag logic)
local is_hld = my_btn:IsDown([btn])           -- Returns true every frame mouse/touch is held inside bounds
local is_hld2 = my_btn:IsHeld([btn])          -- Alias for IsDown

-- Release Detection
local is_rel = my_btn:IsJustReleased([btn])   -- Returns true only on the frame mouse/touch was released

-- Hover & Bounding Queries
local is_hov = my_btn:IsHovered()             -- Returns true if mouse or active touch is inside bounds
local x, y, w, h = my_btn:GetBounds()         -- Absolute screen coordinates and dimensions
local abs_pos = my_btn:GetAbsolutePosition()  -- Vector2 absolute position
local abs_sz = my_btn:GetAbsoluteSize()       -- Vector2 absolute size

-- Event-Driven Callbacks (Zero polling required!)
local click_sub = my_btn:OnClick(function(pos, btn)
    print("Clicked at", pos)
end)
-- Controls: click_sub:Disconnect(), click_sub:remove()

my_btn:OnDown(function(pos, btn) ... end)     -- Fires on press down
my_btn:OnRelease(function(pos, btn) ... end)  -- Fires on release
my_btn:OnHover(function(pos) ... end)         -- Fires when cursor/touch enters
my_btn:OnHoverEnd(function(pos) ... end)      -- Fires when cursor/touch leaves

-- Smooth Window Dragging (Desktop Mouse & Mobile Touch Supported)
local drag = window:MakeDraggable(header_bar, function(new_pos)
    -- Optional callback whenever window is moved
end)
-- Options table format: window:MakeDraggable({ Handle = header, ClampToScreen = true, OnDrag = cb })
-- Controls: drag:SetEnabled(false), drag:Destroy()
```

---

## 7. Animation & Tweening Engine (`tween`)

`penablox RECODE` features a built-in, micro-optimized easing and physics tween engine. It operates with zero heap allocations per frame, running seamlessly on `RenderStepped`. All tweens created by a script are automatically tracked and safely cleaned up on script unload.

### Available Easing Curves
Supported styles: `"linear"`, `"sine"`, `"quad"`, `"cubic"`, `"quart"`, `"quint"`, `"expo"`, `"circ"`, `"back"`, `"elastic"`, `"bounce"`.  
Supported directions: `"in"`, `"out"`, `"inout"`.

### Tween Methods

#### 1. Instant Mathematical Easing
- `tween.ease(alpha, style, direction) -> number`: Returns evaluated eased curve factor `[0..1]`.
- `tween.interpolate(from, to, alpha, [style], [direction])`: Interpolates between two numbers, `Vector2`, `Vector3`, `Color3`, `UDim2`, or `UDim`.

#### 2. Frame-Independent Exponential Smoothing
- `tween.lerp(current, target, speed, dt)`: Smooth, frame-rate independent exponential decay:
```lua
-- Inside a paint / render stepped loop:
current_pos = tween.lerp(current_pos, target_pos, 12, dt)
```

#### 3. Harmonic Spring Physics
Unconditionally stable semi-implicit Euler harmonic springs for ultra-smooth UI animations (switches, tabs, toggles, popups):
- `tween.spring(current, target, velocity, damping, speed, dt) -> (newPos, newVel)`
- `tween.new_spring(initial, [target], [damping], [speed]) -> SpringInstance`:
```lua
local spr = tween.new_spring(0, 100, 0.75, 14)

-- In paint hook:
local cur = spr:Update(dt)
spr:SetTarget(200)
local settled = spr:IsSettled()
```

#### 4. Object Property Animation (`tween.create`)
Animates properties on Drawing objects, custom tables, or Roblox Instances:
```lua
local my_box = Drawing.new("Square")
my_box.Size = vector(100, 40)
my_box.Position = vector(50, 50)
my_box.Color = color(40, 40, 45)

-- Smoothly slide and expand the box:
local anim = tween.create(my_box, 0.35, {
    Position = vector(200, 50),
    Size = vector(160, 40),
    Color = color(70, 130, 240)
}, {
    style = "back",
    direction = "out",
    autoplay = true,
    on_complete = function()
        print("Animation completed!")
    end
})

-- Controls: anim:pause(), anim:play(), anim:cancel(), anim:reset(), anim:reverse()
```

---

## 8. Input & First-Person Cursor Unlocking (`input`)

### First-Person Cursor Unlocking
In Roblox first-person camera mode (`MouseBehavior.LockCenter`), player cursors are locked. The `input` library provides a seamless modal unlocking engine that frees the mouse across the entire screen so players can freely interact with custom Drawing GUIs:

- `input.unlock_cursor()`: Unlocks the mouse cursor, enabling free movement and GUI interaction.
- `input.lock_cursor()`: Restores native mouse lock for gameplay.
- `input.toggle_cursor() -> boolean`: Toggles cursor lock state, returning the new unlocked status (`true`/`false`).
- `input.is_cursor_unlocked() -> boolean`: Checks if the cursor is currently unlocked.
- `input.set_cursor_unlocked(boolean)`: Explicitly sets cursor unlock state.

### Input Polling, Touch & Hit-Testing
- `input.is_just_pressed([btn]) -> boolean`: Returns `true` only on the initial frame mouse/touch was pressed down (single-click detection).
- `input.is_held([btn]) -> boolean`: Returns `true` continuously while mouse/touch is held.
- `input.is_down([btn]) -> boolean`: Alias for `is_held`.
- `input.is_just_released([btn]) -> boolean`: Returns `true` only on the frame mouse/touch was released.
- `input.is_pointer_down([btn]) -> boolean`: Universal pointer check supporting both PC mouse buttons and mobile touch screen inputs.
- `input.is_touch() -> boolean`: Returns `true` if mobile touch screen input is currently active.
- `input.is_hovered(x, y, w, h) -> boolean`: Checks if the mouse cursor or any active touch is inside the specified rectangular region.
- `input.is_clicked(x, y, w, h, [btn]) -> boolean`: Single-click detection inside region (triggers once, does not cycle on hold).
- `input.get_mouse_delta() -> (dx, dy)`: Returns pointer movement delta between frames (works for both mouse drag and touch swipe).
- `input.get_pointer_delta() -> (dx, dy)`: Alias for `get_mouse_delta`.
- `input.get_mouse_position() -> (x, y)`: Screen cursor coordinates in pixels.
- `input.get_pointer_position() -> (x, y)`: Universal pointer coordinates (mouse or primary touch point).
- `input.make_draggable(target, [handleOrOpts], [callback])`: High-level draggable controller with automatic screen boundary clamping.
- `input.is_key_down(key) -> boolean`: Returns `true` if keyboard key (e.g. `"W"`, `"E"`, `"Insert"`) is held.
- `input.is_mouse_down(btn) -> boolean`: Returns `true` if mouse button (`1` = Left/Touch, `2` = Right, `3` = Middle) is held.
- `input.is_typing() -> boolean`: `true` if typing in Roblox chat or a focused UI textbox.
- `input.on_key_press(key, callback)`: Triggers on key press.

---

## 9. Events & Simulation Hooks (`events` / `client`)

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

## 10. Audio & Sound System (`sounds` / `sound`)

The `sounds` (or `sound` / `client.sound`) library gives scripts full control over local 2D and 3D spatial audio playback, as well as the ability to dynamically override the game's native **Hitsound** and **Killsound** with custom Roblox asset IDs or built-in presets.

Overrides automatically reset when the script is unloaded, ensuring no persistent side effects.

### Sound ID Resolution
All sound functions accept:
- **Raw Numbers**: `123456789` -> resolved to `"rbxassetid://123456789"`
- **Numeric Strings**: `"123456789"` -> resolved to `"rbxassetid://123456789"`
- **Full Asset IDs**: `"rbxassetid://123456789"` or `"rbxasset://sounds/..."`
- **Preset Names**: `"skeet"`, `"neverlose"`, `"bell"`, `"bubble"`, `"fatality"`, etc. (case-insensitive)

### API Methods

| Method | Parameters | Description |
| :--- | :--- | :--- |
| `sounds.play(id, [volume], [pitch])` | `id: any, volume?: number, pitch?: number` | Plays a local 2D sound immediately. Returns the created `Sound` instance. |
| `sounds.play_spatial(id, posOrPart, [volume], [pitch], [rollOffMin], [rollOffMax])` | `id: any, target: Vector3 \| BasePart, ...` | Plays 3D positional audio attached to a `Vector3` or `BasePart`. |
| `sounds.play_hitsound([customId], [volume], [pitch])` | `customId?: any, volume?: number, pitch?: number` | Plays the active hitsound (or custom ID if provided). |
| `sounds.play_killsound([customId], [volume], [pitch])` | `customId?: any, volume?: number, pitch?: number` | Plays the active killsound (or custom ID if provided). |
| `sounds.override_hitsound(id, [volume], [pitch])` | `id: any, volume?: number, pitch?: number` | **Overrides in-game hitsound**. Every bullet hit will play this sound instead. Pass `nil` to clear. |
| `sounds.override_killsound(id, [volume], [pitch])` | `id: any, volume?: number, pitch?: number` | **Overrides in-game killsound**. Every kill will play this sound instead. Pass `nil` to clear. |
| `sounds.reset_hitsound()` / `sounds.clear_hitsound_override()` | _none_ | Restores default hitsound to the menu selection. |
| `sounds.reset_killsound()` / `sounds.clear_killsound_override()` | _none_ | Restores default killsound to the menu selection. |
| `sounds.get_hitsound_override()` | _none_ | Returns `{ id = string, volume = number?, pitch = number? }` or `nil`. |
| `sounds.get_killsound_override()` | _none_ | Returns `{ id = string, volume = number?, pitch = number? }` or `nil`. |
| `sounds.get_hitsound_names()` | _none_ | Returns a list of all built-in preset sound names. |
| `sounds.resolve_id(idOrName)` | `idOrName: any` | Resolves a preset name or numeric ID into a valid `"rbxassetid://..."` string. |
| `sounds.presets` | _table_ | Read-only map of preset names to asset IDs (`sounds.presets.skeet`, `sounds.presets.bell`, etc.). |

### Built-in Presets
Built-in preset names:
- `"bell"`
- `"bubble"`
- `"pop"`
- `"ding"`
- `"sparkle"`
- `"laser"`
- `"bip"`
- `"punch"`
- `"crack"`
- `"bite"`
- `"bonk"`
- `"arcade"`
- `"metallic"`
- `"zap"`
- `"click"`
- `"coin"`
- `"glass"`
- `"retro"`
- `"hit"`
- `"skeet"`
- `"neverlose"`
- `"fatality"`

### Example: Custom Hitsound & Killsound Override UI

```lua
local my_tab = menu.new_tab("Audio Mod", "rbxassetid://10734975692")
local sec = my_tab:new_section("Hitsounds", "Left")

local hitsound_names = sounds.get_hitsound_names()
local hs_dropdown = sec:new_combobox("Custom Hitsound", hitsound_names, "skeet", function(chosen)
    sounds.override_hitsound(chosen, 1.0, 1.0)
    client.log("Switched hitsound to preset: " .. tostring(chosen))
end)

local custom_id_box = sec:new_textbox("Custom ID", "Enter Roblox Asset ID", "", function(id)
    if id and #id > 0 then
        sounds.override_hitsound(id, 1.2, 1.0)
        client.log("Applied custom hitsound asset ID: " .. tostring(id))
    end
end)

local pitch_slider = sec:new_slider("Pitch", 50, 200, 100, 0, "%", function(val)
    local cur = sounds.get_hitsound_override()
    if cur then
        sounds.override_hitsound(cur.id, cur.volume or 1.0, val / 100)
    end
end)

-- Play spatial 3D audio on aimbot hit:
client.set_event_callback("aim_hit", function(target, hitbox, damage)
    local head_pos = entity.get_hitbox_position(target, 0)
    if head_pos then
        sounds.play_spatial("sparkle", head_pos, 0.8, 1.2)
    end
end)

client.set_event_callback("shutdown", function()
    sounds.reset_hitsound()
    sounds.reset_killsound()
end)
```

---

## 11. Example Scripts

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

---

### Example 3: Fully Standalone Custom GUI Window (Draggable, Modal Cursor Unlock & Springs)

This complete script demonstrates building an independent standalone GUI entirely through `Drawing.new` and `tween`, featuring first-person cursor unlocking via `input.toggle_cursor()`, window dragging via `input.get_mouse_delta()`, hierarchical child parenting (`.Parent = window`), and spring-animated toggle buttons:

```lua
-- Keybind to open/close custom GUI and unlock mouse in 1st person
local GUI_TOGGLE_KEY = "RightShift"

-- 1. Main Window Container
local window = Drawing.new("Square")
window.Size = vector(320, 240)
window.Position = vector(200, 200)
window.Color = color(20, 22, 28)
window.Rounding = 8
window.BorderColor = color(45, 50, 65)
window.ClipsDescendants = true
window.Visible = false

-- 2. Header Bar (Parented to Window)
local header = Drawing.new("Square")
header.Size = vector(320, 36)
header.Position = vector(0, 0)
header.Color = color(28, 30, 40)
header.Rounding = 8
header.BorderColor = color(55, 60, 80)
header.Parent = window

local title = Drawing.new("Text")
title.Text = "penablox // Standalone Lua GUI"
title.Position = vector(14, 10)
title.Color = color(255, 255, 255)
title.Size = 13
title.Font = 0
title.Parent = header

-- 3. Custom Animated Toggle Component
local toggle_label = Drawing.new("Text")
toggle_label.Text = "Enable Fast Resolver"
toggle_label.Position = vector(20, 60)
toggle_label.Color = color(210, 210, 215)
toggle_label.Size = 13
toggle_label.Parent = window

local toggle_bg = Drawing.new("Square")
toggle_bg.Size = vector(40, 20)
toggle_bg.Position = vector(260, 58)
toggle_bg.Color = color(35, 38, 48)
toggle_bg.Rounding = 10
toggle_bg.BorderColor = color(60, 65, 85)
toggle_bg.Parent = window

local toggle_knob = Drawing.new("Circle")
toggle_knob.Radius = 7
toggle_knob.Position = vector(268, 68)
toggle_knob.Color = color(180, 180, 190)
toggle_knob.Filled = true
toggle_knob.Parent = window

local toggle_state = false
local knob_spring = tween.new_spring(268, 268, 0.72, 16)

-- 4. Window Dragging & Cursor Unlock State
local is_gui_open = false

-- Bind smooth drag handling (Mobile touch + Desktop mouse with auto-clamping)
local drag_handle = window:MakeDraggable(header, function(new_pos)
    -- Optional callback on drag
end)

local function set_gui_open(open)
    is_gui_open = open
    window.Visible = open
    input.set_cursor_unlocked(open) -- Automatically frees mouse in first person!
    
    if open then
        -- Play entrance scale / fade bounce
        window.Position = vector(200, 180)
        local anim = tween.create(window, 0.35, {
            Position = vector(200, 200)
        }, { style = "back", direction = "out" })
        anim:play()
    end
end

-- Hotkey hook to toggle UI
input.on_key_press(GUI_TOGGLE_KEY, function()
    set_gui_open(not is_gui_open)
end)

-- Main Render Loop
client.set_event_callback("paint", function(dt)
    if not is_gui_open then return end

    -- Handle Toggle Click (Single-click edge detection: does NOT cycle when held down!)
    if toggle_bg:IsClicked() then
        toggle_state = not toggle_state
        knob_spring:SetTarget(toggle_state and 292 or 268)
        
        tween.create(toggle_bg, 0.2, {
            Color = toggle_state and color(70, 130, 240) or color(35, 38, 48)
        }):play()
    end

    -- Update animated knob position
    local current_x = knob_spring:Update(dt or 0.016)
    toggle_knob.Position = vector(current_x, 68)
end)

-- Cleanup on unload
client.set_event_callback("shutdown", function()
    input.lock_cursor()
    if drag_handle and drag_handle.Destroy then
        drag_handle:Destroy()
    end
    window:Destroy()
end)
```

---

## 8. In-Game Live Lua Console

penablox RECODE includes a built-in GameSense / Skeet-styled Live Lua Console accessible directly from the menu:
- **Location**: Menu -> **Settings** -> **Lua scripts** -> **"Open lua console"**
- **Toolbar Features**:
  - `Stop output`: Freezes incoming log streaming and autoscroll so you can inspect active output.
  - `Clear`: Flushes all log history and resets line numbers.
  - `Close`: Closes/hides the console window.
- **Copying & Selecting**: The console viewport uses a custom non-editable TextBox (`TextEditable = false`), allowing you to click, drag-select errors, and press `Ctrl+C` to copy error traces with gutter line numbers aligned.
- **Integrated Logging Functions**:
  - `print(...)`: Streams formatted text directly to both Roblox console and Lua Console with prefix `[script.lua]`.
  - `warn(...)`: Streams warnings to the console tagged with `[WARN]`.
  - `client.log(...)`: Logs info entries to the live console.
  - `client.color_log(r, g, b, ...)`: Formats and logs entries to the live console.
  - `client.error(...)`: Logs custom error traces tagged with `[ERROR]`.
  - Automatic compilation & runtime errors, callback crashes, and load/unload notices stream in real-time.

