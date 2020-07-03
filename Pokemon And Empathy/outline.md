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
  - FIXME: Didn't see any explicit zeroing of the table when entering cities.
    How does the zero for 'M end up there if your name is the max length?
    Should check the code for the old man encounter, and/or character creation
- Trainers were only for punctuation characters, so most folks never saw them.
  - FIXME: What actually happens for values that result in a trainer? Is it
    actually a battle or can you catch your rival?

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
  - In Ruby the code looks roughly something like this:
  - ```ruby
    tile = tile_at(9, 9)
    encounter_rate = nil
    if tile.is_grass?
      encounter_rate = @grassEncounterRate
    elsif tile.is_water?
      encounter_rate = @waterEncounterRate
    end

    if encounter_rate
      tile = tile_at(8, 9)
      if tile.is_water?
        perform_encounter(@waterPokemon)
      else
        peform_encounter(@grassPokemon)
      end
    end
    ```
    - We lose something subtle by translating this to a high level language like
      Ruby. This bug could only occur because we're reassigning to `tile`, which
      you would never do normally. When you look at the actual assembly the
      constraints are a bit more clear.
      - We probably need to handwave over this and just go "because assembly".
        The actual reason is because GBZ80 has 7 general purpose registers, and
        they're basically all used all the time. The `cp` instruction clobbers 4
        of them, including the `c` register which is where the subtile gets
        stored
      - FIXME:
        ```
        coord hl, 9, 9
        ld c, [hl]
        ld a, [wGrassTile]
        cp c
        ld a, [wGrassRate]
        jr z, .CanEncounter
        ld a, $14 ; in all tilesets with a water tile, this is its id
        cp c
        ld a, [wWaterRate]
        jr z, .CanEncounter
        ```
        c should be clobbered by the first cp, second cp should never set z. But
        you can encounter pokemon while surfing so something is up


Learning Empathy From Pokemon Blue
====

Have you ever looked at a bug in a video game and wondered why it actually
happens? It's easy to chalk it up to sloppy coding but that's almost never the
case.

In this talk we'll be dissecting a famous exploit from 1999's Pokemon Blue known
as the "Missingno" glitch. We'll look at the details of each of the bugs behind
the seemingly random actions involved in this exploit. We'll look at why these
bugs happened, and the lessons we can apply to our Ruby code more than 20 years
later.

If you've never played or even heard of Pokemon, that's ok! As long as you're
interested in a deep dive into a fascinating set of bugs, this talk is for you.

Details
===

The subject of this talk is developer empathy, not Pokemon. The goal of this
talk is to remove "this is completely broken" from our vocabularies. We'll also
be looking at the importance of encapsulation, and keeping knowledge local and
contained. This talk will take the audience through the series of seemingly
unrelated bugs that together lead to a major exploit. The audience is not
expected to have played Pokemon before attending this talk. As we go through
these bugs, we'll be looking at *why* they may have happened. We'll be focusing
on the constraints the game was developed under, and focus on how these lead to
bugs, not "sloppy coding", laziness, or incompetence.

The glitch that we'll be analyzing is coloquially referred to as the "Missingno"
glitch, as the desired result is an encounter with a glitched out pokemon with
that name. This glitch was well known among kids when this game came out, since
after encountering Missingno, the 6th item in your inventory would have its
quantity increased by 128.

We'll start the talk by demonstrating how the exploit is performed with either a
video or still images. For the members of the audience who have never played
Pokemon, the purpose of this section will be to establish what the game is, and
the effects of the glitch that we're looking at. For the entire audience, the
goal is to establish just how random the actions the player takes are. To give
you an idea of just how random they are, it includes replaying an early game
tutorial and then immediately going up and down the coast of a late game area.
We will ensure this is accessible to folks who have never played the game.

We'll then start to go through each bug in detail. Games in 1999, especially
handheld games, were written in assembly. The hardware constraints were strict
enough that even the overhead of C was too much. There's been an immense effort
in taking the disassembly of Pokemon, labeling things and re-organizing them to
resemble what a developer would have written. This doesn't mean that we're going
to be showing a lot of assembly, but it does mean we have an actual reference of
the source to use to understand these bugs. Any time assembly is shown, we will
be showing equivalent Ruby. Often that leads to extremely un-ideomatic Ruby, and
we'll use those idiosyncrasies to note the hardware limitations of the time.

As we go through the individual bugs, we'll be focusing on a few repeating themes

- These bugs are completely unrelated to each other. There's no way anybody
  would have foreseen these parts of the code interacting with each other.
- More often than not, these bugs were due to side effects of one part of the
  application being unexpectedly observed by another. Working in an object
  oriented language, we have many more tools at our disposal to protect
  ourselves against this.
- There was often some constraint in play that forced clever solutions to
  problems
- It's unlikely that these would have been caught in code review individually
  (and we'll discuss exactly why).

If there is time, we'll cover a few questions that folks who did play the game
when it came out are likely to have:

- What did they do in Pokemon Yellow (an enhanced version that came out a year
  later) to fix this?
- Why did some folks seem to have no problem getting the encounter, but it could
  take hours for others? (This wasn't just folks misunderstanding statistics)
- Why did Missingno sometimes look like a fossil?
- Why were we told not to catch it? What would happen if we did?
- Why was Missingno sometimes called 'M?

The intended audience takeaways are:

- OMG THIS GLITCH WAS SO INTERESTING TO LEARN ABOUT
- Software is never "completely broken". All software has bugs, and they rarely
  come from sloppy coding or laziness. It's more likely you don't know the
  constraints it was developed under.
- While bugs will always happen, focusing on encapsulation and limiting visible
  side effects can keep multiple bugs from interacting with each other to become
  something worse.
- Games of this era are super accessible, and worth digging into the details of
  how these things worked.

This talk was co-authored by two people. Only one of us will be getting on
stage, but we've both put an imense amount of time and energy into this talk.
