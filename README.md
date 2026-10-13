# Playable Jubilee for Marvel Heroes Omega

Jubilee as a full playable hero for **Marvel Heroes Omega 1.52** on an
[MHServerEmu](https://github.com/Crypto137/MHServerEmu) private server. She is built from her team-up model,
so she needs no new character model.

It comes as a **module** for
[MH Launcher](https://github.com/Jazzneo/marvel-heroes-steamdeck-launcher) (1.7.0 or newer). The launcher
puts her into your game and server files, and takes her out again cleanly.

## What you get

- **Powers up to level 30.** There are two power tabs, with Show Starter, Roman Rockets, Superglob,
  Mega Glob, Sparkler Kick (a rollerblade kick), Kaboom and more.
- **Grand Finale** as her ultimate.
- **Firework Skate** as her dodge, with 4 charges.
- **Rollerblades** as her travel power.
- **Skating** in and out when you swap heroes.
- **15 talents and 2 traits.**
- **Her own gear set** (gloves, top and more), with new hand-drawn item icons.

## Install

1. Download `Playable_Jubilee_<version>.zip` from [Releases](../../releases). You don't need to unzip it.
2. In MH Launcher, open **Modules → Install module…** and pick the zip.
3. Make sure the module is turned on, then press **Apply changes**.
4. Press **PLAY**. The launcher copies the new game data to your server too.

## Uninstall or turn off

In **Modules**, either:
- press **Turn off** and then **Apply changes**, or
- press **Remove**.

Either way, the launcher puts back the game files exactly as they were before.

> **Your Jubilee save:** while the module is off, the server can't load Jubilee. It removes her, with her
> level and gear, from your account the next time it saves. The launcher backs up your account before you
> turn her off, so you can restore it later.

## Coming from a hand-installed Jubilee

If you installed an older Jubilee build by hand (the `Jubilee_v0.x` zips), put the original game files back
before you install the module. The launcher checks for this and won't apply Jubilee over a hand-installed
copy, because then turning her off couldn't take her out.

## Known issues

- The team-up version of Jubilee is invisible when you summon her. She appears after a zone change.
- Storm gear can't be equipped on Jubilee, and Jubilee gear can't be equipped on Storm.

## What the module changes

The full list is in [`module/module.json`](module/module.json):

| What | Where |
| --- | --- |
| 133 new prototypes and 5 new blueprints (hero, powers, talents, gear) | `Data/Game/Calligraphy.sip` |
| 200 text lines (names and descriptions) | `Data/Game/Loco/eng.all/…FFFF.string` |
| Her model with hero animations, plus the dodge, kick, travel and swap animations | 6 `UC__*.upk` packages |
| 5 gear icons in SD and HD | `ICO__MarvelUIIcons_SF.upk` and `ICO__MarvelUIIcons_HD_SF.upk` |

The module only **adds** prototypes, blueprints and text. Other heroes aren't changed. The animation packages
it uses are ones the game ships but never loads, so other heroes' powers keep their own animations.

## Module layout

```
module/
  module.json        name, version, and the files it changes
  calligraphy.json   prototypes and blueprints to add, with their IDs and GUIDs
  calligraphy/       the prototype and blueprint files themselves
  files/             whole game files that replace the originals
  textures/          icon pixels (DXT5) written into existing icon packages
  strings/eng.json   English text lines to add
```

To build the release zip, zip the *contents* of `module/` so that `module.json` is at the top level.

## Credits

Made by Jazneo, with help from Claude. Marvel Heroes is © Marvel and Gazillion Entertainment. This is a fan
project for private servers and isn't affiliated with Marvel, Disney or Gazillion.

## Making a new release

1. Change `module/`, raise `"version"` in `module/module.json`, and add a section to `CHANGELOG.md`.
2. Commit and push.
3. On GitHub open **Actions → Release → Run workflow**. Or push a tag that matches the version, such as `v0.6.4`.
4. GitHub Actions builds `Playable_Jubilee_<version>.zip` and publishes it on the Releases page as `v<version>`.
