This page is about word tearing

[Falling Block Episode 4](https://www.youtube.com/watch?v=b-TLpc5oyhg)

# Introduction

"Word tearing", (also called blockstate corruption), occurs when multiple threads concurrently modify block
data stored in the same `long` value, causing unintended results.

This can create any block in the game, or replace blocks with each other within a subchunk.

> Perhaps "bitstorage" or "block storage" corruption are more accurate names, but alas.

## Blockstates in Memory

Each nonempty or previously nonempty subchunk ($16^3$ blocks) stores blocks in a `BitStorage` mapping
each block to a palette id, and a `PalettedContainer`, containing a `Palette` mapping $n$-bit ids to
blockstates.

`BitStorage` packs palette ids in a `long[]` **without padding**, so some blocks' ids get split across
two longs. Access to the array isn't synchronized, so concurrent updates create a race condition where
different threads can overwrite each other's changes. This can result in bits from different blockstates
being combined.

## Blockstate Palette

The palette type used depends on the number of blockstates in the subchunk:
- $[1, 16]$: 4 bits -> `LinearPalette`, an array of `BlockStates` previously present
- $[17, 256]$: 5-8 bits -> `HashMapPalette`, with blockstates previously present
- $[257, 16^3]$: 13 bits -> Registry (`GlobalPalette`), using `Block.STATE_REGISTRY.get(id)`

The blockstate registry `Block.STATE_REGISTRY` assigns each blockstate a 13-bit id given by
`(blockId << 4) | metadata`. `blockId` is obtained from `Block.REGISTRY`, which stores all blocks
in a map with 9-bit ids. Thus the blockstate registry uses 13-bit ids.

Blockstates are not removed from a palette unless it upsizes. A palette never downsizes while the
chunk is loaded, it must be reloaded from disk with fewer blockstates.

Upsizing the palette when an async observer line is running in the subchunk is [generally a bad idea](./async-line.md#random-palette-corruptions).

## Race Condition Example

When the game updates a block in `BitStorage#set`, it first reads out the first long containing the
block's id, updates and writes it, and then updates the second long if the block's id is split in two.

```text
0b 0000 010010000: minecraft:water[level=0]
0b 0100 100010000: minecraft:anvil[damage=0,facing=south]

0b 0100 010010000: minecraft:command_block[conditional=false,facing=down]

Block     |-----1-----| |-----2------|
13-bit id 0000000000000 000010010 0000
long      --------A-------------| |-B-
```
- Block 1's palette id is stored entirely in `long A`.
- Block 2's palette id (water) is split across two consecutive `long`s, lower bits `010010000` in `A`
  and upper bits `0000` in `B`.
- Suppose the following occurs when Thread 1 updates Block 1 (e.g. a blinking async observer line)
  and Thread 2 updates Block 2 to anvil:

| Step | Thread 1         | Thread 2          | Result                                          |
| ---- | ---------------- | ----------------- | ----------------------------------------------- |
| 1    |                  | reads A           |                                                 |
| 2    | reads original A |                   |                                                 |
| 3    |                  | writes modified A | Block 2 lower bits updated to anvil `100010000` |
| 4    | writes stale A   |                   | Block 2 lower bits reset to water   `010010000` |
| 5    |                  | writes B          | Block 2 upper bits updated to anvil `0100`      |

Now Block 2 has the old lower bits of water and new upper bits of anvil, resulting in an unintended
block id for command block.

### Word Resetting

If a block is stored in the middle of a long and the long gets updated on two threads at the same time,
the updated block may get reset back to the original.
For example, this can result in air being reset to block 36, or observers being reset to powered.

# Registry Word Tearing

The registry palette contains block ids for every blockstate in the game. Word tearing here can be
used to create blockstates that weren't originally present in the subchunk, including all the
[unobtainables](./unobtainables.md).

To get the registry palette, we need $>256$ blockstates in the subchunk. Cheap blocks for increasing
palette size include slabs, stairs, fence gates, doors, trapdoors, glazed terracotta, redstone wire,
pistons, etc. in different rotations.

# Hashmap Word Tearing

Word tearing with a Hashmap Palette allows you to replace a block with another block already present
in the palette, or with an unused id to be assigned later. This is mostly useful for falling block
swaps in the [Generic Method](./falling-block/generic-method.md), bedrock mining swap setups, or
outright duping blocks.

- $[17, 32]$ blockstates: 5 bits
- $[33, 64]$ blockstates: 6 bits
- $[65, 128]$ blockstates: 7 bits
- $[129, 256]$ blockstates: 8 bits

Word tearing is possible on 5, 6, or 7 bit hashmap palettes, since they don't divide 64 bits in a long.
7 is the most useful since it leaves us room to avoid palette upsizing.

When a chunk is loaded, it will assign ids to each blockstate in `xzy` order, starting with `0: air`.
Any new blockstates are assigned the next available id.

# Word Tearing Contraption

To word tear two specific blocks, the first block must be **directly** replaced by the other, while
an async observer line is blinking and resetting blocks stored in the same long.
Steps like block 1 -> air -> block 2, will word tear with air separately. The probability of word
tearing is likely hardware-dependent; with an async observer line blinking at max speed it could be
between $1/100$ and $1/500$.

Be sure to periodically check the end of the async observer line, it may be stuck in the powered
state due to [word resetting](#word-resetting). Break and replace the observer, or move it with a
piston to fix it.

## Gravity block mixers

This is the fastest way to do word tearing, by dropping gravity blocks into fire, air, water, or lava.
**Make sure Instant Fall is OFF**.

To do this, we put the gravity block over a block to be pushed and pulled by a piston.
1. When the piston is activated, the block under the gravity block is replaced by air.
2. An ITT observer line activates and updates the gravity block several hundred times in the tick,
  creating falling block entities.
3. When each falling block gets ticked, it replaces the gravity block with air.
4. If there is water or lava, they flow into the block. An observer line pointing into a dispenser with
  flint and steel can be used to place fire.
5. The entity then lands and places its block, to be deleted by the next falling block.

Word tearing can occur at any of these `setBlockState` calls, unintended ones can create byproducts.
Word tearing air into flowing liquids or flowing liquids into still liquids usually isn't noticeable.

`setBlockState` calls for block replacement:
- **gravity block -> air -> gravity block**
- gravity block -> air -> flowing_water -> **water -> gravity block**
- gravity block -> air -> flowing_lava -> **lava -> gravity block**
- gravity block -> air -> **fire -> gravity block**

## Piston mixers

Piston clocks can be used to replace movable blocks with air, or block 36 (`piston_extension`) with
movable blocks.
A simple 6 tick clock can be used, or a more complicated 3 tick clock.

Only sticky piston head and sticky piston base turn into sticky block 36. All blocks pushed by sticky
pistons are normal block 36.

`setBlockState` calls for block replacement:
- block -> air -> piston_extension[facing=north,type=normal] -> block

We can prevent word tearing between some setBlockStates by locking the async line when it's not needed,
preventing any byproducts from being created. This is mostly useful for automatic generic method setups.

## Miscellaneous

- Armor stands pushed by a piston clock can perform `water -> frosted_ice[age=0]` word tearing.
  The frosted ice should have enough light to melt instantly, otherwise it will crash on ITT.
  Armor stands can be stacked on top of each other to speed this up.
  - Additionally, `frosted_ice[age=3] -> water` word tearing can occur when it melts in the overworld.
  - In the nether, `frosted_ice[age=3] -> air -> flowing_water -> water` word tearing can occur, be
    sure to recreate the water from covered source blocks.
- Manually placing or breaking blocks can word tear with air, fire, water, lava.
- Manually placing blocks into water or lava that can be destroyed by them can word tear
  liquid -> block, and flowing liquid -> block.
- Tallgrass and double tallgrass can be replaced with any block.
- Grass -> Grass Path, dirt -> farmland.

## Word Tearing Locations

In these tables, $a:b$ means the block's **lower** $a$ bits are stored in the first long, and **upper**
$b$ bits are stored in the second long. The `-` blocks are stored entirely within the long.

E.g. Command Block has id `0100 010010101`, placed at $x=2, z=14$ ($9:4$). Then long 1 has
`...101010010` and long 2 has `0010...`, when printed out with the `/palette` command. The bit
printout is reversed from `0b100010010101` because higher bits are stored later in the array.

Observers in the same long, in `-x` direction reset the lower bits ($a$), observers in `+x` direction
reset the upper bits ($b$). (Wrap around at chunk boundary in `z`).

Long boundaries are the same for every `y`-level, since 64 bits evenly divides 256 blocks per `y`-level.

### 13 bits

| Z \ X | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 0 | - | - | - | - | 12:1 | - | - | - | - | 11:2 | - | - | - | - | 10:3 | - |
| 1 | - | - | - | 9:4 | - | - | - | - | 8:5 | - | - | - | - | 7:6 | - | - |
| 2 | - | - | 6:7 | - | - | - | - | 5:8 | - | - | - | - | 4:9 | - | - | - |
| 3 | - | 3:10 | - | - | - | - | 2:11 | - | - | - | - | 1:12 | - | - | - | - |
| 4 | - | - | - | - | 12:1 | - | - | - | - | 11:2 | - | - | - | - | 10:3 | - |
| 5 | - | - | - | 9:4 | - | - | - | - | 8:5 | - | - | - | - | 7:6 | - | - |
| 6 | - | - | 6:7 | - | - | - | - | 5:8 | - | - | - | - | 4:9 | - | - | - |
| 7 | - | 3:10 | - | - | - | - | 2:11 | - | - | - | - | 1:12 | - | - | - | - |
| 8 | - | - | - | - | 12:1 | - | - | - | - | 11:2 | - | - | - | - | 10:3 | - |
| 9 | - | - | - | 9:4 | - | - | - | - | 8:5 | - | - | - | - | 7:6 | - | - |
| 10 | - | - | 6:7 | - | - | - | - | 5:8 | - | - | - | - | 4:9 | - | - | - |
| 11 | - | 3:10 | - | - | - | - | 2:11 | - | - | - | - | 1:12 | - | - | - | - |
| 12 | - | - | - | - | 12:1 | - | - | - | - | 11:2 | - | - | - | - | 10:3 | - |
| 13 | - | - | - | 9:4 | - | - | - | - | 8:5 | - | - | - | - | 7:6 | - | - |
| 14 | - | - | 6:7 | - | - | - | - | 5:8 | - | - | - | - | 4:9 | - | - | - |
| 15 | - | 3:10 | - | - | - | - | 2:11 | - | - | - | - | 1:12 | - | - | - | - |

### 7 bits

Most [Generic Method](./falling-block/generic-method.md) setups use $1:6$ word tearing.
Since these positions only occur at $x=9$, and falling block swaps should occur more than 8 blocks
from [clustered chunks](./chunk/chunk-hashmap.md#cluster-chunks), we want to ensure the northeast
2x2 chunks are not clustered for $(9, 0)$ and $(9,4)$, and the southeast 2x2 chunks are not clustered
for $(9, 8)$ and $(9, 12)$.

| Z \ X | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 0 | - | - | - | - | - | - | - | - | - | 1:6 | - | - | - | - | - | - |
| 1 | - | - | 2:5 | - | - | - | - | - | - | - | - | 3:4 | - | - | - | - |
| 2 | - | - | - | - | 4:3 | - | - | - | - | - | - | - | - | 5:2 | - | - |
| 3 | - | - | - | - | - | - | 6:1 | - | - | - | - | - | - | - | - | - |
| 4 | - | - | - | - | - | - | - | - | - | 1:6 | - | - | - | - | - | - |
| 5 | - | - | 2:5 | - | - | - | - | - | - | - | - | 3:4 | - | - | - | - |
| 6 | - | - | - | - | 4:3 | - | - | - | - | - | - | - | - | 5:2 | - | - |
| 7 | - | - | - | - | - | - | 6:1 | - | - | - | - | - | - | - | - | - |
| 8 | - | - | - | - | - | - | - | - | - | 1:6 | - | - | - | - | - | - |
| 9 | - | - | 2:5 | - | - | - | - | - | - | - | - | 3:4 | - | - | - | - |
| 10 | - | - | - | - | 4:3 | - | - | - | - | - | - | - | - | 5:2 | - | - |
| 11 | - | - | - | - | - | - | 6:1 | - | - | - | - | - | - | - | - | - |
| 12 | - | - | - | - | - | - | - | - | - | 1:6 | - | - | - | - | - | - |
| 13 | - | - | 2:5 | - | - | - | - | - | - | - | - | 3:4 | - | - | - | - |
| 14 | - | - | - | - | 4:3 | - | - | - | - | - | - | - | - | 5:2 | - | - |
| 15 | - | - | - | - | - | - | 6:1 | - | - | - | - | - | - | - | - | - |

## Selected Recipes

Refer to the [blockstate list](../resources/block_state_id_list_by_masa.txt). You can search `0b___`
for bits at the beginning at `___:` for bits at the end of an id.

| result | original | replace | location $a:b$ | observer position | comment |
| --- | --- | --- | --- | --- | --- |
| `command_block` | `water` | `anvil` | 9:4 | `-x` | `level` of water determines command block direction |
| `end_portal_frame` | `air` | `dragon_egg` | 6:7, 7:6 | `-x` | observer in `+x` works for the reverse |
| `chain_command_block` | `fire` | `end_rod` | 9:4 | `+x` | |
| `repeating_command_block` | `grass` | `grass_path` | 6:7, 7:6, 8:5 | `-x` | |
| `structure_block` | `fire` | `concrete_powder` | 6:7 | `-x` | |
| `structure_void` | `water` | `frosted_ice` | 8:5 | `+x` | can be in `-x` in the overworld for frosted_ice -> water |
| `end_gateway` | `water` | `frosted_ice` | 7:6 | `-x` | can be in `+x` in the overworld for frosted_ice -> water |
| `mob_spawner` | `fire` | `sand` | 7:6 | `+x` | |
| `mob_spawner` | `air` | `redstone_wire` | 6:7 | `-x` | observer in `+x` works for the reverse |
| `barrier` | `iron_trapdoor` | `air` | 5:8 | `+x` | use a piston clock |
| `bedrock` | `water` | `redstone_wire` | 7:6 | `+x` | |
| `bedrock` | `air` | `redstone_wire` | 7:6, 8:5 | `+x` | |
| double_stone_slab [seamless=false, variant=wood_old] | `carpet[color=magenta]` | `water` | 10:3, 11:2 | `-x` | or air, observer in `+x` works for `flowing_water` reverse |
| Seamless Stone Slab | `carpet[color=silver]` | `water` | 10:3, 11:2 | `-x` | can also use other carpet colors |
| Seamless Sandstone Slab | block 36 | `carpet[color=cyan]` | 8:5 | `+x` | can also use other color carpets |
| `monster_egg` | `water` | brown_mushroom_block [variant=all_outside] | 6:7, 7:6 | `-x` | `level` of water determines monster egg variant |
| `tnt[explode=true]` | piston_extension facing=up, type=normal | `tnt[explode=false]` | 2:10, 3:9 | `-x` | do not punch |
| `water[level=0]` | `end_stone` | `air` | 8:5 | `-x` | |
| `flowing_water[level=0]` | `air` | `flowing_water[level=2]` | 2:10, ..., 7:6 | `-x` | turns into source water |
| `water[level=0]` | `lava` | `anvil[damage=0,facing=south]` | 6:7, 7:6 | `+x` | |
| `tallgrass[type=dead_bush]` (shrub) | `air` | `tallgrass[type=tall_grass]` | 1:11, ... 4:9 | `-x` | use shears |
| `log[axis=none,variant=oak]` | piston_extension facing=west, type=normal | `log[axis=z,variant=oak]` | 3:10 | `-x` | similar setups for other bark blocks |
