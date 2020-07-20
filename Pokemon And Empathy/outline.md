What is this glitch we're looking at?
--

- The glitch duplicates any item in your inventory 128 times
- This glitch didn't work for everybody, and everyone experienced it
  differently. We'll go into why.
- It was often called the "rare candy" glitch, since that was what most people
  copied.
  - Explain what rare candy is
- Here's how the glitch worked (video demo?)
  - Talk to old man
  - Do weedle fight
  - Fly to cinnabar island
    - Probably have to mention what fly is briefly
  - Go to the east coast
  - Surf up and down (do not go too far right!) until you encounter missingno
    - Don't catch it! (Or so we were told)
  - Run or knock it out
  - You may also encounter unexpected pokemon at really high levels, or
    glitched trainer battles.
  - After the battle, the 6th item in your inventory will have its quantity
    increased by 128.
  - If you had hall of fame data, it'll probably be corrupted
- A few interesting notes
  - Everyone seemed to experience the glitch slightly differently. A handful
    of folks saw glitched trainer battles, but it was rare. Some folks also
    saw missingno without bugged out sprites, but that was also rare.
  - Allegedly catching missingno would corrupt your save, but that didn't
    happen for everyone
  - All the pokemon you could encounter were levels that were impossible to
    achieve normally

How does any of this actually work
--

- The glitch is a combination of a half dozen completely unrelated bugs, that
  happen to have the desirable effect of item duplication
- Probably start by discussing the constraints of GB dev, 8k of RAM, reasons
  to avoid storing things on the cartridge unless absolutely necessary
- What order should these be presented in?
  - One obvious answer is in the order we do the glitches, not sure how well
    this flows though
    - Main benefit here is that the talk has a natural start by showing a video
      of the glitch
  - Maybe start from the goal and work backwards?
    - If we do this do we start by demonstrating the glitch? Do we show a video
      of each piece?
    - Probably still want to leave why the item count change happens for the
      end, it's relatively mundane by itself and feels like a good payoff at the
      end
    - Present the glitch as a goal: "If we can fight a MissingNo. we will get
      128 rare candies"
    - This is probably where we discuss what MissingNo. is, that there are
      multiple versions, maybe mention 'M here?
    - So our goal is to encounter a MissingNo, let's look at how Pokemon Blue
      determined encounters
    - Present Route 20, maybe mention safari zone glitch, reframe the goal as if
      we could somehow load an encounter table with missingno into the grass
      table, we can use route 20 to fight it
    - Somewhere in there we need to mention that the grass table isn't
      overwritten when the zone doesn't have one
    - Now we can talk about the old man glitch, start talking about how names
      worked
    - Maybe at this point we fast forward back to encountering it and we can
      start discussing all the different variants
      - Mention glitched trainer battles at this point too? Definitely not
        spending a lot of time on that
    - Finally we show why the item count changes
    - Maybe mention hall of fame data corruption here too
    - Definitely mention why its sprite gets that weird shape

Old man glitch
--

Half of this needs to get split. Not sure how much I want to get into how
pokemon are stored, what missingno is, etc. The relevant bit for *this glitch*
is that your name goes in the encounter table 
- When you talk to the "old man" in Viridian, the cutscene where he shows you
  how to catch pokemon isn't actually a cutscene. It's a scripted sequence of
  events that happen in the engine. This means that the game temporarily needs
  to change the player's name to "old man", which means the player's name has
  to be stored somewhere. Since this happens in a city, and cities don't have
  any encounters, the developer chose to store it where the encounter table is
  normally stored.
- This by itself wouldn't do anything. In order to exploit this fact, you need
  to somehow get a wild pokemon encounter without the game overriding your
  name with a real encounter table.
- This does, however, explain why this glitch was different for everyone. It
  changed based on what was in your name!
- The list of wild pokemon you can encounter is 20 bytes. The level of a
  pokemon, followed by its ID. There's one set for grass, another for water.
  Fishing isn't tied to zone in Gen 1.
- The text encoding used had 80 as an "end of name" marker, and actual
  printable characters started at 127. This is why the pokemon are such high
  levels. The even characters of your name would be the level of the pokemon,
  and the odd characters would be the ID.
- If your name was an even number of characters, or if it contained a space,
  G, H, J, M, S, :, ], a, b, c, m, o, p, v, w, x, or y in the spot for an odd
  character, you can encounter MissingNo. If you used a custom name, you can
  encounter a similar pokemon called `'M`, which is a separate glitch pokemon,
  but will still cause the same item duplication.
  - Even number of characters works because 80 is one of the MissingNo IDs, so
    if the end of name byte is in an ID slot, it will cause MissingNo.
  - These characters all correspond to a pokemon that doesn't actually exist,
    causing the glitched out portrait, and the item duplication
- You can't encounter 'M if you used a preset name, but all preset names have
  a character that would lead to MissingNo. Because all custom names can
  encounter 'M, this means all players would be able to perform the item
  duplication glitch.
  - 'M is ID 0. Since the name length is smaller than the encounter table,
    you'll always end up with trailing zeroes, so you can always encounter 'M
    at either level 80 or level 0 depending on if your name had an even or odd
    number of characters.
  - Preset names can't encounter 'M, because they're all stored as one giant
    block, separated only by byte 80 which is "end of name". e.g.
    RED[80]ASH[80]JACK[80]NEW NAME[80]
  - Custom names could always encounter 'M due to 0s at the end of their names.
    Names in general were 11 bytes, but the player could only input 7. Bytes
    9-11 would always be 0.
- You could also get glitched trainer battles, but those only happened for
  punctuation characters, so most folks never saw them.

Other ways to load missingno into the encounter table
--

- Trades and link battles store the other player's name in the same spot as the
  grass encounter table.
  - Notably, all NPCs that trade pokemon are named TRAINER, so their name is
    `<TRAINER>@@@@@@@@@@` (0x5D followed by 0x80 10 times), which gives an
    encounter table consisting entirely of level 80 missingnos. There is an NPC
    trade in the Cinnabar lab, which is likely how this glitch was originally
    discovered

Getting the glitched encounter
--

- The next step is to actually get an encounter. To do this we immediately fly
  to Cinnebar island and surf on the east coast. This is specifically chosen
  because to exploit the two bugs in this section we need a tile that is:
  - In an area without a grass encounter table (any city or water route)
  - Accessible without entering an area that has a grass encounter table
  - Has a coast where the land is on the left side
- We need to avoid going to any area with a grass encounter table, since if we
  did that our name would be overwritten with real pokemon
- This limits us to cities, since the only way we can get there without going
  through a route which has grass encounters is to use fly.
- When you enter an area, the game checks to see if there's an encounter table
  to use, and if there isn't it skips the copy.
  - We could maybe talk about why they would have written it this way. I think
    it was most likely an optimization for size. The code size caused by having
    this conditional is a lot smaller than the number of zero bytes they would
    have needed to include to unconditionally do the copy. 20 bytes for every
    area without grass (cities and water routes), 20 bytes for every area
    without water (cities, some routes)
- The coast we go to is technically part of route 20, which has an encounter
  table for water pokemon, but not grass. This puts us in a state where our name
  occupies the encounter table for grass, and there's a real encounter table for
  water.
- This still isn't enough to get a glitched encounter though. Our name is in the
  grass table, but there's no grass for us to walk through. Even if there was,
  the encounter rate for grass right now is zero!
- The last bug that lets us get an encounter is why we specifically need a coast
  tile with land on the left side. If you've ever played Pokemon, you know that
  it's a grid of tiles, but from the game's perspective the grid is actually 1/4
  the size. So the player occupies a square of 4 tiles.
  - When the game looks to see if an encounter can happen, it looks at the
    bottom right subtile, and continues if it's grass or water (this is why
    those tiles with a flower in the bottom right corner can't have wild
    encounters). When it checks what encounter table to use, it looks at the
    bottom left.
  - In Rust the code looks roughly something like this:
  - ```rust
    let tile = tile_at(9, 9);
    let encounter_rate = if tile.is_grass() {
      current_area.grass_encounter_rate
    } else if tile.is_water() {
      current_area.watter_encounter_rate
    } else {
      return;
    };

    let tile = tile_at(8, 9)
    if tile.is_water() {
      perform_encounter(current_area.water_pokemon)
    } else {
      peform_encounter(current_area.grass_pokemon)
    }
    ```
    - We lose something subtle by translating this to a high level language like
      Rust. This bug could only occur because we're reassigning to `tile`, which
      you would never do normally. When you look at the actual assembly the
      constraints are a bit more clear.
      - We probably need to handwave over this and just go "because assembly".
        The actual reason is because GBZ80 has 7 general purpose registers, and
        they're basically all used all the time. The address of the tile was
        stored in hl, and its value was loaded into c. These end up getting
        re-used to hold the pokemon being encountered, and the second coord is
        loaded into a. It looks like if a and c were swapped in
        TryDoWildEncounter#next, the reload wouldn't have been needed. However
        this may have been a constraint of which instructions work with which
        registers, it may have been a desire to not have register allocation be
        reasoned about over too much code, it may have been intentional, or just
        a mistake.
        - I think it's due to instruction limitations. `ld a, [addr]` is GB
          specific, and AFAICT only allows the `a` register for arbitrary
          addresses, but you can specifically load `[hl]` into any register
    - THIS SLIDE IS GOING TO BE HARDEST TO MAKE ACCESSIBLE
    - DO NOT ASSUME THE AUDIENCE KNOWS WHAT A REGISTER IS, DON'T TRY TO DO THIS
      BY EXPLAINING HOW OPCODES WORK
    - In the assembly, the loads are 45 lines apart

What even is a MissingNo
--

MissingNo vs 'M
--

Results of the encounter
--

- There were two visible effects of the encounter with either MissingNo or 'M
  - Your hall of fame data (if present) would be corrupted
  - The 6th item in your inventory would have its quantity increased by 128
  - If you caught it, Exeggutor would be marked as seen.
- Things that people believed were true that were not true
  - Your game would not be saved as a result of the encounter
  - Catching the pokemon was completely safe and would not corrupt your save
    - There is a very specific issue if you tried to withdraw a level 0 'm from
      the PC that would freeze your game. This would not happen with MissingNo,
      or with 'M of any other level. Since all players with a custom name could
      encounter a level 0 'M, and since you almost certainly had a party of 6
      pokemon at this point which meant 'M would be sent to the PC if you caught
      it, I suspect this is why people got the idea it was unsafe to catch.
- Despite MissingNo's sprite very clearly being garbage data, MissingNo actually
  does have appropriate entries, and is not the 
- Looking up the base stats and sprites for missingno end up underflowing into
  trainer party data. Missingno's stats end up coming from 

Random hardware/asm notes
---

- The stack was 286 bytes
- a, f, and hl couldn't really be used as general purpose registers
  - Most operations stored their output in a, f was read only except for
    loading af from the stack, you could only load addresses stored in hl
    to registers other than a
- Explain to the audience as "you get four global mutable variables that are 1
  byte each, write the largest handheld video game ever made"

Random pokemon notes
---

- The anime and games have used the same voice actress for Pikachu since 1997,
  and she has appeared in more episodes of the anime than anybody else

Things that we almost certainly won't have time for in the talk, but maybe do a followup blog post about
---

- Most of the places where there are large gaps of unused memory are in sram. I
  suspect this was because using sram was far more difficult than wram since
  you have to deal with bank switching
- It looks like they ultimately failed at avoiding increasing the cartridge
  size. Nearly half of the ROM on the cartridge is unused (banks 0x2D-0x3F are
  unused, they would have needed to fit in 0x20 banks to go down a size). I
  can't find a disassembly or ROM map for green, but I'll bet green did fit on
  the next size down. You're not going to rewrite your whole app to take
  advantage of the doubled size when the goal is to localize it.
