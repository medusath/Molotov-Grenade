# Molotov Grenade — AMX Mod X Plugin for CS 1.6

AMX Mod X plugin for Counter-Strike 1.6 that adds a **Molotov cocktail** (Terrorists) and **Incendiary Grenade** (Counter-Terrorists) as a standalone grenade in a dedicated slot.

---

## Requirements

- AMX Mod X 1.9.0+
- ReGameDLL 5.14.0.453+
- ReAPI, HamSandwich, FakeMeta, XS
- Modified `delta.lst` — body field expanded from 8 to 32 bits:
  ```
  DEFINE_DELTA( body, DT_INTEGER, 32, 1.0 )
  ```

---

## Features

### Core Mechanics
- Adds a new grenade type into its own slot (slot 4), without replacing any existing weapon.
- On impact the grenade detonates and spawns a fire zone that persists for a configurable duration.
- **Progressive damage** — damage increases the longer a player remains inside the fire. The multiplier scales from `1.0` up to `molotov_damage_max_mult`. A quick dash through barely scratches; standing still is lethal.
- **Cylinder-shaped hit zone** — standard GoldSrc sphere checks allow fire to bleed through floors and ceilings. The plugin uses a flat cylinder instead: horizontal XY range plus a vertical window of `[-64, +120]` units. Players on a different elevation are unaffected.
- **Gradual ignition** — with `molotov_spread_enable` the fire begins with a reduced area and grows to full radius halfway through the burn time.
- **Traceline visibility** — damage is only applied when a clear line of sight exists between the fire origin and the player. Walls and obstacles block the effect even inside the radius.

### Purchase
- Console and chat commands: `molotov`, `mol`, `incendiary`, `inc`, `/molotov`, `/mol`, `/inc`, `buy molotov`
- Adjustable price, per-round purchase cap, round-based unlock
- Optional automatic issue at round start via `molotov_equip_access`
- Flag-based access control
- Honors `mp_buytime` and buy zone boundaries

### Team-Specific Behavior

Grenade identity is written once at the moment it is given and **persists on the entity itself** throughout its lifecycle — weapon slot, airborne projectile, fire zone. If a CT picks up a dropped TT molotov it retains its original type. Every visual and gameplay decision reads from the entity, never from the holder's team.

|                      | Terrorists               | Counter-Terrorists          |
|----------------------|--------------------------|-----------------------------|
| Name                 | Molotov                  | Incendiary Grenade          |
| View model           | v_molotovgrenade.mdl     | v_incendiarygrenade.mdl     |
| Player model         | p_molotovgrenade.mdl     | p_incendiarygrenade.mdl     |
| World model          | w_molotovgrenade.mdl     | w_incendiarygrenade.mdl     |
| Burning wick (throw) | Yes                      | No                          |
| Hold flash           | Yes                      | No                          |
| Ground fire sprite   | molotov_fire_ground.spr  | incendiary_fire_ground.spr  |
| Spark bursts         | No                       | Yes (`TE_SPARKS`, 12.5 % per tick) |
| Killfeed tag         | [ᴍᴏʟᴏᴛᴏᴠ]               | [ɪɴᴄ]                       |
| HUD sprite file      | weapon_molotov.txt       | weapon_incendiary.txt       |

### Interactions
- **Smoke suppression** — a smoke grenade thrown near the fire extinguishes it. The suppression area remains active for `molotov_smoke_duration` seconds, neutralizing any grenade that passes through — effectively sealing off an approach.
- **Water contact** — a TT molotov shatters on water; a CT incendiary detonates. Grenades that sink can still be retrieved by nearby players.
- **Skybox removal** — optionally destroys the grenade on skybox collision to prevent out-of-bounds throws.
- **Air-hit damage** — a grenade that strikes a player mid-flight deals `molotov_hit_player` damage on the spot, before the fire zone appears.

### Visuals
- Animated multi-part floor fire model (`molotov_fire_floor.mdl`) that fades out smoothly 1-2 seconds before the fire expires
- Ground sprites placed with **randomized yaw** and correctly oriented along surface normals — they sit flush on ramps and stairs instead of clipping
- Debris particles (`TE_SPRITE`) scattered **evenly across the entire burn area** using `sqrt(random)` radial correction to avoid center-heavy clustering
- Scorch decals stamped on the ground after the fire dies
- Glass shard gibs on molotov impact
- Animated point light (`TE_DLIGHT`) with a flickering color channel that simulates a warm fire glow

---

## CVars

| CVar | Default | Description |
|------|---------|-------------|
| `molotov_buy_access` | `""` | Access flags. Empty = available to everyone. |
| `molotov_equip_access` | `0` | Issue automatically at round start (1 = on). |
| `molotov_check_buyzone` | `1` | Enforce buy zone requirement (1 = on). |
| `molotov_cost` | `800` | Price. |
| `molotov_buy_limit` | `1` | Purchases allowed per round. -1 = no cap. |
| `molotov_limit_round` | `0` | Earliest round the grenade becomes available (0 = always). |
| `molotov_hit_player` | `40` | Direct-hit damage. |
| `molotov_killfeed` | `1` | Display killfeed tag. |
| `molotov_radius` | `150` | Fire zone radius in units. |
| `molotov_throwtime` | `3.0` | Fuse time — seconds before mid-air auto-detonation. |
| `molotov_duration` | `8` | Total burn time in seconds. |
| `molotov_damage_mode` | `0` | 0 = hostiles only, 1 = hostiles + thrower, 2 = everyone. |
| `molotov_damage_radius_mode` | `1` | 1 = uniform damage, 2 = falloff by distance and armor. |
| `molotov_damage_time` | `0.5` | Interval between damage ticks (seconds). |
| `molotov_damage_value` | `5` | Base damage per tick. |
| `molotov_damage_max_mult` | `3.0` | Peak multiplier for sustained fire exposure. |
| `molotov_effect_num` | `4` | Debris sprite count per tick. |
| `molotov_smoke_touch` | `1` | Allow smoke to extinguish fire. |
| `molotov_smoke_owner` | `models/w_smokegrenade.mdl` | Smoke grenade world model path. |
| `molotov_smoke_duration` | `20.0` | Smoke suppression zone lifespan (seconds). |
| `molotov_water_touch` | `1` | Extinguish on water contact. |
| `molotov_sky_force` | `1` | Destroy on skybox contact. |
| `molotov_spread_enable` | `0` | Gradual ignition mode. |
| `molotov_spread_radius` | `50` | Extra units added to radius on ignition spread. |

A config file is generated automatically at `addons/amxmodx/configs/plugins/molotov_grenade.cfg` on the first server start.

---

## Natives

```pawn
native GiveUserMolotov(id);       // Hand a molotov to the specified player
native bool:IsUserHasMolotov(id); // Query whether the player is carrying one
```

Example:
```pawn
if (!IsUserHasMolotov(id))
    GiveUserMolotov(id);
```

---

## Compile-time Options

| Define | Purpose |
|--------|---------|
| `#define ALLOW_CUSTOMNADE` | Compatibility with third-party equipment plugins that occupy the `WEAPON_GLOCK` slot (e.g. Health Nade). When enabled, the molotov is moved to `WEAPON_TMP` — the TMP is sacrificed in exchange. Uncomment to activate. |

---

## Installation

1. Compile `molotov_grenade.sma` → place `molotov_grenade.amxx` in `addons/amxmodx/plugins/`
2. Copy `molotov_grenade.txt` → `addons/amxmodx/data/lang/`
3. Copy `sprites/grenade/` → server `cstrike/sprites/grenade/`
4. Copy `models/grenade/` → server `cstrike/models/grenade/`
5. Copy `sound/weapons/grenade/` → server `cstrike/sound/weapons/grenade/`
6. Add `molotov_grenade.amxx` to `addons/amxmodx/configs/plugins.ini`
7. Update `delta.lst`: `DEFINE_DELTA( body, DT_INTEGER, 32, 1.0 )`

---

## Credits

Thanks to: fantom, Garey, Denzer, Shel, OLOF, SISA, wopox1337, fl0wer, Xelson, steelzzz

---
---

# Молотов / Зажигательная граната — AMX Mod X для CS 1.6

Плагин добавляет в Counter-Strike 1.6 полноценную зажигательную гранату в отдельный слот: **коктейль Молотова** для Террористов и **зажигательную гранату** для Контр-Террористов.

---

## Требования

- AMX Mod X 1.9.0+
- ReGameDLL 5.14.0.453+
- ReAPI, HamSandwich, FakeMeta, XS
- Модифицированный `delta.lst` — поле body расширено с 8 до 32 бит:
  ```
  DEFINE_DELTA( body, DT_INTEGER, 32, 1.0 )
  ```

---

## Возможности

### Основные механики
- Новый тип гранаты в собственном слоте (слот 4), без замены какого-либо штатного оружия.
- При ударе о поверхность граната детонирует и порождает зону горения, которая держится настраиваемое время.
- **Прогрессивный урон** — чем дольше игрок находится в пламени, тем сильнее его обжигает. Множитель плавно растёт от `1.0` до `molotov_damage_max_mult`. Короткая перебежка через огонь почти безвредна; бездействие — смертельно.
- **Цилиндрическая зона поражения** — обычная сферическая проверка GoldSrc пробивает межэтажные перекрытия. Плагин использует плоский цилиндр: горизонтальная дистанция по XY плюс вертикальное окно `[-64, +120]` юнитов. Игроки на другой высоте не задеты.
- **Постепенное разгорание** — при включённом `molotov_spread_enable` зона горения стартует уменьшенной и достигает полного радиуса к середине времени горения.
- **Трейс видимости** — урон проходит только при прямой видимости между точкой возгорания и игроком. Стены и препятствия блокируют поражение даже внутри радиуса.

### Покупка
- Команды в консоли и чате: `molotov`, `mol`, `incendiary`, `inc`, `/molotov`, `/mol`, `/inc`, `buy molotov`
- Настраиваемая цена, ограничение покупок за раунд, номер раунда разблокировки
- Автовыдача при старте раунда через `molotov_equip_access`
- Контроль доступа по флагам
- Учитывает `mp_buytime` и границы зоны покупки

### Поведение по командам

Тип гранаты фиксируется в момент выдачи и **сохраняется на самой энтити** на всём её жизненном пути — слот инвентаря, летящий снаряд, зона огня. Если КТ подберёт выброшенный молотов ТТ, он сохранит свой исходный тип. Все визуальные и игровые решения читаются из энтити, а не из команды текущего владельца.

|                        | Террористы               | Контр-Террористы            |
|------------------------|--------------------------|-----------------------------|
| Название               | Молотов                  | Зажигательная граната       |
| Модель от первого лица | v_molotovgrenade.mdl     | v_incendiarygrenade.mdl     |
| Модель игрока          | p_molotovgrenade.mdl     | p_incendiarygrenade.mdl     |
| Мировая модель         | w_molotovgrenade.mdl     | w_incendiarygrenade.mdl     |
| Горящий фитиль (бросок)| Да                       | Нет                         |
| Вспышка при удержании  | Да                       | Нет                         |
| Спрайт огня на полу    | molotov_fire_ground.spr  | incendiary_fire_ground.spr  |
| Всплески искр          | Нет                      | Да (`TE_SPARKS`, 12,5 % за тик) |
| Метка в киллфиде       | [ᴍᴏʟᴏᴛᴏᴠ]               | [ɪɴᴄ]                       |
| Файл спрайтов HUD      | weapon_molotov.txt       | weapon_incendiary.txt       |

### Взаимодействия
- **Подавление дымом** — дымовая граната, брошенная рядом с огнём, гасит его. Зона подавления остаётся активной `molotov_smoke_duration` секунд и нейтрализует любую гранату, пролетающую сквозь неё — фактически перекрывая проход.
- **Контакт с водой** — молотов ТТ разбивается о воду; зажигательная КТ детонирует. Затонувшие гранаты могут быть подобраны ближайшими игроками.
- **Столкновение со скайбоксом** — опционально уничтожает гранату, предотвращая броски через границы карты.
- **Попадание в воздухе** — граната, угодившая в игрока на лету, наносит `molotov_hit_player` урона мгновенно, ещё до появления зоны горения.

### Визуальная часть
- Анимированная составная модель огня на полу (`molotov_fire_floor.mdl`), которая плавно угасает за 1-2 секунды до окончания горения
- Напольные спрайты со **случайным разворотом yaw**, корректно ориентированные по нормали поверхности — на рампах и лестницах лежат ровно, а не уходят в текстуру
- Частицы дебриса (`TE_SPRITE`), **равномерно рассеянные по всей площади горения** с радиальной коррекцией `sqrt(random)`, чтобы избежать сгущения в центре
- Следы ожогов (декали) на полу после затухания огня
- Осколки стекла при ударе молотова
- Точечный динамический свет (`TE_DLIGHT`) с мерцающим цветовым каналом, имитирующий тёплое свечение пламени

---

## Кварсы

| Квар | По умолчанию | Описание |
|------|-------------|----------|
| `molotov_buy_access` | `""` | Флаги доступа. Пусто = доступно всем. |
| `molotov_equip_access` | `0` | Выдавать автоматически при старте раунда (1 = да). |
| `molotov_check_buyzone` | `1` | Требовать зону покупки (1 = да). |
| `molotov_cost` | `800` | Стоимость. |
| `molotov_buy_limit` | `1` | Допустимое число покупок за раунд. -1 = без ограничений. |
| `molotov_limit_round` | `0` | Раунд, начиная с которого граната доступна (0 = всегда). |
| `molotov_hit_player` | `40` | Урон при прямом попадании. |
| `molotov_killfeed` | `1` | Отображать метку в киллфиде. |
| `molotov_radius` | `150` | Радиус зоны горения в юнитах. |
| `molotov_throwtime` | `3.0` | Время полёта до автоподрыва (секунды). |
| `molotov_duration` | `8` | Общее время горения (секунды). |
| `molotov_damage_mode` | `0` | 0 = только противники, 1 = противники + бросивший, 2 = все. |
| `molotov_damage_radius_mode` | `1` | 1 = одинаковый урон, 2 = затухание по дистанции с учётом брони. |
| `molotov_damage_time` | `0.5` | Интервал тиков урона (секунды). |
| `molotov_damage_value` | `5` | Базовый урон за тик. |
| `molotov_damage_max_mult` | `3.0` | Пиковый множитель при продолжительном нахождении в огне. |
| `molotov_effect_num` | `4` | Число спрайтов дебриса за тик. |
| `molotov_smoke_touch` | `1` | Дымовая граната тушит огонь. |
| `molotov_smoke_owner` | `models/w_smokegrenade.mdl` | Путь к мировой модели дымовой гранаты. |
| `molotov_smoke_duration` | `20.0` | Время жизни зоны подавления дымом (секунды). |
| `molotov_water_touch` | `1` | Гасить при контакте с водой. |
| `molotov_sky_force` | `1` | Уничтожать при столкновении со скайбоксом. |
| `molotov_spread_enable` | `0` | Режим постепенного разгорания. |
| `molotov_spread_radius` | `50` | Дополнительные юниты радиуса при разгорании. |

Файл конфигурации создаётся автоматически в `addons/amxmodx/configs/plugins/molotov_grenade.cfg` при первом запуске сервера.

---

## Нативы

```pawn
native GiveUserMolotov(id);       // Выдать молотов указанному игроку
native bool:IsUserHasMolotov(id); // Узнать, есть ли у игрока молотов
```

Пример:
```pawn
if (!IsUserHasMolotov(id))
    GiveUserMolotov(id);
```

---

## Опции компиляции

| Дефайн | Назначение |
|--------|------------|
| `#define ALLOW_CUSTOMNADE` | Совместимость со сторонними плагинами снаряжения, занимающими слот `WEAPON_GLOCK` (например Health Nade). При включении молотов переносится на слот `WEAPON_TMP` — TMP при этом жертвуется. Для активации раскомментируйте строку. |

---

## Установка

1. Скомпилировать `molotov_grenade.sma` → поместить `molotov_grenade.amxx` в `addons/amxmodx/plugins/`
2. Скопировать `molotov_grenade.txt` → `addons/amxmodx/data/lang/`
3. Скопировать `sprites/grenade/` → `cstrike/sprites/grenade/` на сервере
4. Скопировать `models/grenade/` → `cstrike/models/grenade/` на сервере
5. Скопировать `sound/weapons/grenade/` → `cstrike/sound/weapons/grenade/` на сервере
6. Добавить `molotov_grenade.amxx` в `addons/amxmodx/configs/plugins.ini`
7. Обновить `delta.lst`: `DEFINE_DELTA( body, DT_INTEGER, 32, 1.0 )`

---

## Благодарности

Спасибо: fantom, Garey, Denzer, Shel, OLOF, SISA, wopox1337, fl0wer, Xelson, steelzzz
