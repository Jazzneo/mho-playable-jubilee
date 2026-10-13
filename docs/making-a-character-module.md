# How to make your own character module

This guide shows how Playable Jubilee was made, so you can turn another character into a playable hero for
Marvel Heroes Omega 1.52 on MHServerEmu. At the end you package it as a module that
[MH Launcher](https://github.com/Jazzneo/marvel-heroes-steamdeck-launcher) (1.7.0 or newer) can install and
remove.

It assumes you're comfortable with Python and with poking at binary files. You don't need Unreal Engine or
the game's source.

---

## 1. How a hero is put together

A playable hero lives in three places. A module can change all three:

| Part | Lives in | Read by | What it holds |
| --- | --- | --- | --- |
| **Game data** | `Data/Game/Calligraphy.sip` | the server **and** the game | The hero, costume, powers, power screen, talents, traits, gear, item slots, numbers. These are "prototypes". |
| **Text** | `Data/Game/Loco/eng.all/*.string` | the game | Names and descriptions that prototypes point to by string ID. |
| **Look** | `UnrealEngine3/MarvelGame/CookedPCConsole/*.upk` | the game | The model, animations, power effects and icons. |

The server only needs the game data. MH Launcher copies `Calligraphy.sip` to the server every time you press
PLAY, so the server and the game always agree.

## 2. Pick your two starting points

You need two things:

1. **A model that already has animations.** Team-up heroes are the best source, because each one already has a
   full model with attack animations. Jubilee uses her team-up package, `UC__MarvelTeamUp_Jubilee_SF.upk`.
   NPCs and bosses can work too, but they usually have fewer animations.
2. **A template hero** whose game data you copy and change. Jubilee started as a copy of **Storm**: her hero
   entry, costume, power screen layout, talent grid and gear slots. Pick a hero who plays roughly the way your
   character should.

Copying a working hero and changing it is much easier than building prototypes from scratch, because every
field the game expects is already there.

## 3. Unpack the game data

`Calligraphy.sip` is a pack file. Each file inside is stored in **LZ4** blocks. The two files that matter
most are:

- **`Calligraphy/Prototype.directory`**: every prototype the game knows. After an 8-byte header (`PDR` plus a
  version byte, then the entry count as a `u32`), each entry is:
  ```
  u64 id, u64 guid, u64 blueprint_id, u8 flags, u16 length, path (backslashes)
  ```
- **`Calligraphy/Blueprint.directory`**: the blueprints, which say which fields a prototype type has. Each
  entry is:
  ```
  u64 id, u64 guid, u8 flags, u16 length, path
  ```

The prototype files themselves (`*.prototype`, `*.defaults`, `*.curve`, and so on) are binary. MHServerEmu's
source has a full reader for this format, under `MHServerEmu.Games/GameData/Calligraphy`. Use it to see how
fields are laid out, then write a small reader and writer in your own tool. You only need to read a file,
change some fields and write it back.

## 4. Make the game data

Jubilee's game data has these parts. Make each one by copying the template hero's version and changing it:

| Part | Copied from | What to change |
| --- | --- | --- |
| Hero (`Entity/Characters/Avatars/Shipping/<Name>.prototype`) | Storm | Name, description, model (`UnrealClass`), icons, starting costume, travel power, power progression, talents |
| Costume | Storm's costume | `CostumeUnrealClass` (the model), icons, `UsableBy` (your hero) |
| Powers (`Powers/Player/<Name>/…`) | The team-up's powers, or similar powers | Name, icon, cooldown, cost, damage curve, animation package (`PowerUnrealClass`) |
| Power screen | Storm's `PowerProgressionTables` | Which power unlocks at which level, its tab and its position |
| Dodge | The game's `DashPowerTemplate` | Charges, speed, animation |
| Travel power | A sprint-style travel power | Animation and start delay |
| Talents and traits | Storm's talents, re-pointed | Which of your powers each one boosts |
| Gear (optional) | Storm's gear | Its own item types, slots, inventories and icons. Needed so your hero and the template hero can't wear each other's gear. |

Set `IsStarterAvatar` to 1 on the hero so it's unlocked on every account.

### IDs and GUIDs: get these right

Every prototype has an **ID** worked out from its path, and a **GUID**:

```python
import zlib

def hash_id(s):
    b = s.lower().encode()
    return ((zlib.adler32(b) | (zlib.crc32(b) << 32)) - 1) & 0xFFFFFFFFFFFFFFFF

def prototype_id(path):          # "Powers/Player/Jubilee/Boom.prototype"
    return hash_id(path.replace(".", "?").replace("/", "."))
```

- A `.defaults` prototype (only needed when you add a new blueprint) has the same ID as its blueprint, and uses
  flags `3`.
- The **GUID** can be any random 64-bit number that isn't already used in the directory. **Saves refer to
  heroes and items by GUID.** Once you release a version, never change an existing GUID. If you do, players
  lose the hero and her gear.
- **Add new entries at the end** of the directory and keep their order the same from one version to the next.
- Other prototypes use flags `2`. Blueprints use flags `1`.

### Text

Prototypes don't contain text. They hold a **string ID**, and the text lives in the `.string` files. Pick new
IDs that aren't used yet. The top two bits of the ID choose the file, so an ID starting with `0xC…` goes into
`eng.all_FFFFFFFFFFFFFFFF.string`. Jubilee uses IDs from `0xC1D3A00000000101` up, counting in steps of
`0x10`.

## 5. Model and animations

### The game only loads packages it shipped with

This rule matters more than any other in this guide. **The game won't load a `.upk` file that it didn't come
with.** A new file in `CookedPCConsole` is ignored without any error. Editing `AssetPackageCache.bin` doesn't
help either, because the game ignores those changes.

The way around it is to **reuse packages the game already has but never uses**, and put your content inside
them. Jubilee reuses these:

| Package | Used for |
| --- | --- |
| `UC__MarvelTeamUp_Jubilee_SF.upk` | Her model, with hero animations added (idle, run, hit and death) |
| `UC__PowerRogue_JubileeBoom_SF.upk` | Firework Skate (dodge) |
| `UC__PowerElektra_Kick_SF.upk` | Rollerblades travel power |
| `UC__PowerElektra_Kick2_SF.upk` | Sparkler Kick |
| `UC__PowerAvatarSwapIn_SF.upk` / `SwapOut` | Skating in and out when you swap heroes |

To find unused packages, check the game's `Launch.log` while you play. Packages that never show up there and
that no prototype points to are safe to reuse.

### Animations

- Team-up models are missing some animations a hero needs, such as idle, run, hit and death. Copy them from
  a hero with a similar skeleton. Jubilee's came from Black Widow: 95 of Jubilee's 118 bones also exist in
  Black Widow's skeleton. Move bone positions over as *Jubilee's base pose + Black Widow's movement*, not
  as raw values, or the body will stretch.
- Remove **AnimNotifies** from animations you copy. They refer to sounds and effects in the other hero's
  package, and the log fills with "bad AnimNotify" errors.
- Animations are stored as `acf_float96now` / `akf_variablekeylerp`. You can cut out a section of an
  animation, change its speed (`RateScale`) or paste in root motion without re-encoding anything else.
- A **T-pose** when you swap to the hero means an animation the hero uses is missing from its package.

### Team-up and hero share one model

If the hero uses the team-up's model, summoning the team-up version shows it **invisible** until you change
zones. Jubilee has this known issue.

## 6. Icons

Item and power icons live in `ICO__MarvelUIIcons_SF.upk` (SD, 40×40 **DXT5**) and
`ICO__MarvelUIIcons_HD_SF.upk` (HD, 128×128 DXT5). You can't add new icons, so pick existing icons your
hero's items don't need and paint over them:

- **SD:** put the raw DXT5 pixels in the module's `textures/` folder. The launcher writes them into whatever
  icon package the player has, compressed or not, and works alongside other mods.
- **HD:** ship the whole package in `files/`.

## 7. Package the module

A module is a folder, or a zip of one, with `module.json` at the top:

```
module.json        name, version and the files it changes
calligraphy.json   new prototype and blueprint directory entries (id, guid, blueprint, flags, path)
calligraphy/       the prototype and blueprint files, at their path inside Calligraphy/
files/             whole game files that replace the originals (models, animations, HD icons)
textures/          SD icon pixels (DXT5)
strings/eng.json   {"<16-hex-digit string id>": "text"}
```

```json
{
  "format": 1,
  "id": "playable-yourhero",
  "name": "Playable Your Hero",
  "version": "1.0.0",
  "author": "You",
  "description": "Shown on the Modules tab.",
  "game": "Marvel Heroes Omega 1.52.0.1700",
  "files": ["UnrealEngine3/MarvelGame/CookedPCConsole/UC__MarvelTeamUp_YourHero_SF.upk"],
  "textures": [{"package": "ICO__MarvelUIIcons_SF.upk", "texture": "Item_Art329", "file": "textures/Item_Art329.dxt5"}],
  "calligraphy": "calligraphy.json",
  "strings": {"eng": "strings/eng.json"}
}
```

Jubilee's own files in [`module/`](../module) are a complete, working example. For every field, see the
launcher's
[module format guide](https://github.com/Jazzneo/marvel-heroes-steamdeck-launcher/blob/main/docs/modules.md).

Raise `version` every time you change something. The launcher rebuilds the game files when the version
changes.

## 8. Test it

Always test with the **original game files**, not on top of a hand-installed copy. The launcher refuses to
apply a module over files that already contain it.

1. In MH Launcher, open **Modules → Install module…**, pick your folder or zip, then press **Apply changes**.
2. Press **PLAY**, make the hero, and check:
   - every power at the right level
   - the dodge and travel power
   - swapping in and out
   - talents
   - gear, both equipping it and that other heroes can't wear it
3. Check the game's log for errors. It's in the launcher's game prefix:
   `drive_c/users/steamuser/Documents/My Games/Marvel Heroes/MarvelGame/Logs/Launch.log`
4. **Turn the module off** and press Apply. The game files should be back exactly as they were. Turn it on
   again and the hero should still be there.

> While a hero module is turned off, the server can't load that hero. It removes the hero from the account the
> next time it saves. The launcher backs up the account first, but test on an account you don't mind losing.

## Checklist before you release

- [ ] The hero is a copy of a template hero, with name, model, icons and costume changed
- [ ] New prototypes are added at the end, with GUIDs that never change once released
- [ ] Every animation and effect is inside a package the game already has
- [ ] No "bad AnimNotify" or missing-package errors in `Launch.log`
- [ ] The module turns off and on cleanly with MH Launcher
- [ ] `version` is raised, and `CHANGELOG.md` is updated
