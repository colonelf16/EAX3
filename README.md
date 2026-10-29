# Project EAX

Personal-use Windows tool that recreates the Sound Blaster Live! 24-bit External control panel
(Surround Mixer, EAX Settings, Speaker Settings, Graphic Equalizer) without the sound card.

It mixes your microphone and system audio ("What U Hear"), applies EAX room effects, CMSS 3D,
EQ and bass processing, and plays the result into **VB-Cable's CABLE Input**. Pick
**CABLE Output** as the microphone in Discord or a game.

![The four panels](docs/panels.png)

## Build

1. Open `ProjectEAX.sln` in Visual Studio 2026 (C++ desktop + .NET desktop workloads).
2. Pick **Debug** or **Release**, platform **x64**, and build.
3. Run `build\<Configuration>\Creative EAX Settings.exe` (UI) or `EaxProto.exe` (console prototype).

| Project | What it is |
| --- | --- |
| `src/EaxEngine` | C++ DLL: WASAPI capture/render and the effects chain. C API in `include/eax_engine.h`. |
| `src/EaxProto` | C++ console prototype and test tool for the engine. |
| `src/CreativeEaxSettings` | "Creative EAX Settings": C# WinForms UI styled like the Creative panels. Forms open in the VS designer. |

The reverb, preset, EQ and mixing code in `src/EaxEngine/third_party/octopus` comes from
OctopusEAX (MIT, see its `LICENSE`). The skin bitmaps, cursors and icons in `src/CreativeEaxSettings`
come from the Creative software; keep this repository private.

## Using it

- The app opens on **EAX Settings**. The buttons along the bottom open the other panels; each panel
  closes on its own. With every panel closed the app keeps running in the notification area (EAX
  icon) so the effects keep working, like the card's driver did: click the icon to reopen, right-click
  > Exit (or ▾ > Exit in any panel) to quit. Starting the program again brings EAX Settings back.
- **Speaker Settings > Speaker Selection**: pick your **Headphone device** (system audio is
  captured from it) and your **Microphone**. Devices with "CABLE" in their name are never listed,
  so the output can't feed back into itself.
- **Surround Mixer**: *Wave* = system audio, *Line-In/Mic-In* = microphone, *What U Hear* = the
  level of the finished mix into CABLE Input. MIDI and CD Audio are kept for the look only.
- **EAX Settings**: room preset (Haunted Cavern is the classic one) and Effects Amount; CMSS 3D widening.
  Unticking *Enable Audio Effects* cuts the reverb at once (no tail); changing rooms lets the old one ring out.
  Music and game sound get the whole room OctopusEAX plays program audio in on its monitor (with
  Haunted Cavern: the stereo cavern with cascade/centre and WetPeak voicing, the slap echo and the
  core). A fixed trim per amount keeps the song's total loudness where it was (-6.6 dB x amount/100).
  Measured on a song-like signal, the echo after it stops is now 4.1 dB under the music at amount 30
  (was 9.8) and 1.5 dB at 60 (was 4.5). The mic's room is unchanged.
- **Speaker Settings > Speaker Selection** picks Headphone, 2.1, 4.1, 5.1, 6.1 or 7.1. The Speaker Test
  picture shows that layout and Channel/Noise play each speaker in turn on the headphone device.
- **Bass Management** works like the card's: with *Bass Redirection* on, a 4th-order Linkwitz-Riley
  crossover sends everything below the crossover frequency to the subwoofer, whose level is
  *Subwoofer Volume* (the same setting as the mixer's Subwoofer slider). Off, stereo music has no
  subwoofer signal. Because CABLE Input is stereo, the subwoofer is folded back into both channels.
- **Surround Mixer > Advanced** follows the speaker selection, like the original: Headphone shows
  Center / Rear Center / Subwoofer and the Balance/Fader pad; the speaker layouts show one volume per
  speaker. The mix is split over those virtual speakers and folded back to stereo, so every slider
  changes the sound; at 100% (and balance/fader centred) nothing changes.
- **Speaker Settings > Microphone**: *Mic Boost* (0..+26 dB) works like OctopusEAX's mic fader, with
  its preamp drive and bass lift. The mic is always opened in Raw mode, so Windows' own mic effects
  (auto gain, noise suppression) stay out; a mic that doesn't offer Raw mode is opened normally.
- **Hot input from an outside boost stays clean.** The mic may arrive far above full scale (a boost
  in Equalizer APO, VoiceMeeter or another float program) and is kept whole. Before Mic Boost, an
  input trim (`engine/InputTrim.h`) brings it back under -1 dBFS with one fixed gain set by the
  loudest peak of the last 10 s. It doesn't ride the voice up and down: it only goes lower when a
  peak louder than anything in the last 10 s arrives (a few milliseconds ahead, so nothing clips),
  and back up (3 dB/s) once that peak has left the window, e.g. when the outside boost is turned down.
  Normal mic levels pass untouched, so Mic Boost gets what a normal mic gives it and sounds the same.
  Gain that a driver adds in whole numbers (16/24-bit) is cut off at full scale before any program
  gets it, so that kind can't be repaired afterwards.
- **Haunted Cavern lowers your dry voice as the amount goes up** (OctopusEAX's "step back", so the
  echo can cover it): -2.2 dB at 40, -5.9 dB at 60, -10.9 dB at 80, -16.5 dB at 100. It is a fixed
  cut per amount, not a level reaction.
- **Graphic Equalizer**: 7 bands, level, presets (save your own with the disk button).
- Settings are saved to `%APPDATA%\ProjectEAX\settings.json`; engine state changes go to `engine.log` there.
- First start without VB-Cable shows a setup prompt: it opens the download page or runs an installer
  you pick (only if it is signed by VB-Audio) with a Windows permission prompt. Nothing is bundled.

## Discord, VoiceMeeter and system audio

System audio is everything that plays on the headphone device, Discord included: friends in a call
hear their own voices come back. To avoid that, set Discord's output to a different device than the
Headphone device, or turn *Wave* down.

**VoiceMeeter is left out automatically.** VoiceMeeter usually plays your mic on the headphones (its A1
output), so system audio would carry your voice a second time, a little late: hollow and phasey,
with the background doubled, and worse the more gain you add outside the app. While VoiceMeeter plays
on the headphone device, the engine captures each other program separately with process loopback
and leaves VoiceMeeter out (`src/EaxEngine/io/SessionLoopback.cpp`); the status line says so.

- Programs that play *into* VoiceMeeter (its VAIO / AUX VAIO inputs, often Windows' default output,
  so Chrome, games...) reach the headphones only through VoiceMeeter's stream. They are captured
  themselves (a process stream records a program on any device), so they still get the effects.
- Windows system sounds are not captured in this mode.
- Process loopback is documented from Windows 10 build 20348 and works on 19041+ (tested on 19045).
  If Windows refuses it, the app captures the whole device and the status line says so.
- Why not `PROCESS_LOOPBACK_MODE_EXCLUDE_TARGET_PROCESS_TREE`: process loopback isn't tied to one
  device, so "everything except VoiceMeeter" would also capture this app's own output into CABLE
  Input (a feedback loop). Instead each program on the headphone device gets its own INCLUDE stream.

## Console prototype and checks

```
EaxProto --list                       devices as the UI would offer them, CABLE Input, presets
EaxProto --selftest                   test tones measured on CABLE Output: levels, mute, centre/balance,
                                      bass redirection, effects-off cut, mic boost, +20 dBFS outside boost,
                                      soft clip, loopback,
                                      VoiceMeeter left out (a copy of EaxProto stands in for it)
EaxProto --mic PodMic --seconds 10    live run: mic + system audio -> effects -> CABLE Input
"Creative EAX Settings" --snapshot <folder>       render every window/tab (and every speaker layout) to PNG
"Creative EAX Settings" --simulate-no-cable       show the VB-Cable setup prompt without installing anything
```
