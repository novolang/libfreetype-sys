# libfreetype-sys

FreeType is a software library that renders text. It reads a font file,
finds the glyph for a character, scales that glyph to a size, and turns
its outline into a bitmap. It is documented in the
[FreeType 2 API reference](https://freetype.org/freetype2/docs/reference/index.html),
and it is the engine behind the text on most Linux desktops, on
Android, and in most PDF and browser rendering stacks. This package
declares fifty-four of that library's entry points to novo-lang, one
declaration each.

Every function here is a declaration of a function in FreeType. The
package contains no logic of its own, and it does nothing without the C
library installed. The fifty-four entry points are the ones a program
needs to open a font, measure it, load a glyph and rasterise it. The
section "What is not included" says what a program cannot do with them
alone.

## What it is

A **face** is one font inside a font file. A file may hold several — a
TrueType collection holds a whole family — and a face is chosen by
index. A face carries the facts that do not depend on a size: the
number of glyphs, the family and style names, the design units per em,
and the bounding box of every glyph in the font.

A **glyph** is one drawn shape. It is not a character: a character code
is mapped to a glyph index through a **character map**, one glyph may
serve several characters, and glyph index 0 is the substitute glyph a
font shows for a character it does not have.

**Design units** are the coordinate system a font is drawn in. The
`units_per_EM` value says how many of them span one em, and 1000 and
2048 are the usual numbers. A size turns design units into pixels.

A **size** is a face scaled to a resolution. Setting it fills in the
size metrics: the pixels per em in each direction, the two scaling
factors, and the scaled ascender, descender and line height.

The **glyph slot** is where a loaded glyph lands. A face has exactly
one, every load overwrites it, and it holds the glyph's metrics, its
outline, and its bitmap once the glyph has been rendered.

An **outline** is the glyph's shape: a list of points, a tag per point
saying whether it is on the curve or a control point, and a list of
indices saying where each contour ends. Rendering turns an outline
into a **bitmap**, which is a row count, a width, a pitch in bytes and
a buffer.

**Fixed point** is how FreeType carries fractions. A 26.6 number is a
value multiplied by 64, and it is the unit of scaled coordinates and
advances. A 16.16 number is a value multiplied by 65536, and it is the
unit of scaling factors and matrices.

## Install

```
novo pkg add libfreetype-sys
```

Adding the package does not install the C library. On Debian and Ubuntu
the library and its headers come from the system package
`libfreetype-dev`:

```
sudo apt install libfreetype-dev
```

On macOS the Homebrew formula is `freetype`. On other systems the
library builds from the FreeType source.

## Example

A character measured and rasterised at sixteen pixels:

```novo ignore
use libfreetype

fn main() [io, fs, ffi]
    let slot = ptr.alloc_word()
    if libfreetype.ft_init_freetype(slot) != 0
        println("FreeType did not start")
        return
    let lib = ptr.read_word(slot)

    let fslot = ptr.alloc_word()
    if libfreetype.ft_new_face(lib, "/usr/share/fonts/x.ttf", 0, fslot) != 0
        println("that file is not a font")
        return
    let face = ptr.read_word(fslot)

    // The face's own facts, read out of its record by offset.
    println("${ptr.read_str(ptr.read_word(face + 40))} has "
            + "${ptr.read_word(face + 32)} glyphs")

    // Sixteen pixels tall, then the glyph for "A".
    let _ = libfreetype.ft_set_pixel_sizes(face, 0, 16)
    let _ = libfreetype.ft_load_char(face, 65, 0)
    let gs = ptr.read_word(face + 152)

    // The advance is the fifth of the eight metrics, in 26.6 units.
    println("advance ${ptr.read_word(gs + 48 + 32) / 64} pixels")

    // Rasterise into the slot's bitmap and read its size.
    let _ = libfreetype.ft_render_glyph(gs, 0)
    let bitmap = gs + 152
    let rows = ptr.read_word(bitmap) & 4294967295
    let width = (ptr.read_word(bitmap) >>> 32) & 4294967295
    println("${width} by ${rows} pixels of grey")

    let _ = libfreetype.ft_done_face(face)
    let _ = libfreetype.ft_done_freetype(lib)
    ptr.free(fslot)
    ptr.free(slot)
```

The example is fenced as an illustration rather than a compiled block
because `novo doc` compiles the blocks in documentation comments and not
the ones in this file. The same calls are in
`tests/libfreetype_tests.nv`, where the numbers are asserted.

## What the package contains

| Module | Contents |
| --- | --- |
| `libfreetype` | Every entry point, in nine groups: the library, the face, the character maps, the sizes, the glyph loader and renderer, the outline helpers, the bitmap helpers, the SFNT tables, and the fixed-point arithmetic. |

The nine groups and their sizes:

| Group | Entry points | What it does |
| --- | --- | --- |
| The library | 5 | Starts and stops FreeType, reports its version, and sets a driver property. |
| The face | 5 | Opens a font from a path or from memory, counts its references, and names it. |
| Character maps | 7 | Selects a map, and turns character codes and glyph names into glyph indices. |
| Sizes | 5 | Sets the size four ways, and sets the transform applied to every later load. |
| Loading and rendering | 5 | Loads one glyph, rasterises it, and answers the advance and the kerning. |
| Outlines | 12 | Creates, copies, moves, transforms, thickens and measures an outline. |
| Bitmaps | 5 | Initialises, copies, converts, thickens and releases a bitmap record. |
| SFNT tables | 4 | Reads a raw table out of the font file, and the name records in it. |
| Fixed point | 6 | Multiplies, divides and rounds in FreeType's own 16.16 arithmetic. |

## How to choose an entry point

`FT_New_Face` is for a font on disk, and `FT_New_Memory_Face` for
bytes already in memory. The memory loader does not copy the bytes, so
the caller keeps them alive for the life of the face; in exchange it
touches no file and carries no `[io]` effect.

`FT_Set_Char_Size` is for a size in points, which needs a resolution.
`FT_Set_Pixel_Sizes` is for a size already in pixels.
`FT_Request_Size` is for the cases neither covers: fitting a glyph's
real dimension, its bounding box or its cell into a number of pixels.
`FT_Select_Size` is for a face that carries fixed bitmap strikes rather
than outlines.

`FT_Load_Char` is `FT_Get_Char_Index` followed by `FT_Load_Glyph`. Use
`FT_Load_Glyph` when the glyph index came from somewhere else, which is
what a text shaper answers.

`FT_Get_Advance` is for a program that is laying text out and does not
want the outline. It is faster than a load, and it is the only call
here that answers a measurement without touching the glyph slot.

`FT_Outline_Get_CBox` encloses the control points and costs almost
nothing. `FT_Outline_Get_BBox` solves the curves and answers the
smallest box the glyph really fits in. For a glyph with no curves they
answer the same four numbers.

## The rules a user needs

1. **`FT_Init_FreeType` comes first and `FT_Done_FreeType` comes
   last.** Nothing else in the library may be called outside the pair,
   and releasing the library invalidates every face, outline and
   bitmap created under it.
2. **A pointer is an `Int`, and zero is null.** Every handle the C
   library returns arrives as the address it returned.
3. **An out-parameter is the address of a caller-owned slot.** Every
   call that produces a handle writes it into one, because the return
   value is the error code. `ptr.alloc_word` reserves a slot,
   `ptr.read_word` reads it back, and `ptr.free` releases it.
4. **A field narrower than eight bytes is masked out of a word.**
   `ptr.read_word` loads eight bytes from an address. Mask the width
   of the field with `&`, and shift with `>>>` when the field does not
   begin the word.

   | C type | How to read it at address `a` |
   | --- | --- |
   | `unsigned short` | `ptr.read_word(a) & 65535` |
   | `unsigned int` | `ptr.read_word(a) & 4294967295` |
   | `signed int` | `ptr.read_word(a) as i32` |
   | a field 2 bytes into the word | `(ptr.read_word(a) >>> 16) & 65535` |

5. **An error code is a small positive number**, and 0 is success.
   These are the codes this package's entry points give.

   | Code | Name | What it means |
   | --- | --- | --- |
   | 0 | — | success |
   | 1 | `FT_Err_Cannot_Open_Resource` | the file is not there or cannot be read |
   | 2 | `FT_Err_Unknown_File_Format` | the file opened and is not a font |
   | 6 | `FT_Err_Invalid_Argument` | the index, the glyph or the property is not there |
   | 8 | `FT_Err_Invalid_Table` | the face is not an SFNT face |
   | 19 | `FT_Err_Cannot_Render_Glyph` | the glyph slot holds nothing that can be rasterised |
   | 23 | `FT_Err_Invalid_Pixel_Size` | the face has no outlines and no strike of that size |
   | 35 | `FT_Err_Invalid_Face_Handle` | the face carries no strikes at all |
   | 142 | `FT_Err_Table_Missing` | the SFNT table is not in this font |

6. **`FT_Error_String` answers 0 on most builds.** The table is
   compiled in only with `FT_CONFIG_OPTION_ERROR_STRINGS`, and the
   Debian and Ubuntu packages are built without it. A 0 means this
   build has no table, not that the code is unknown.
7. **A face's facts are fields of its record.** These are the offsets,
   for the `x86_64` System V layout FreeType 2.13 compiles to.

   | Offset | Width | Field |
   | --- | --- | --- |
   | 0 | 8 | `num_faces`, the fonts in the file |
   | 8 | 8 | `face_index` |
   | 16 | 8 | `face_flags` |
   | 32 | 8 | `num_glyphs` |
   | 40 | 8 | `family_name`, the address of a string |
   | 48 | 8 | `style_name`, the address of a string |
   | 72 | 4 | `num_charmaps` |
   | 80 | 8 | `charmaps`, the address of an array of addresses |
   | 104 | 32 | `bbox`, four design-unit values: xMin, yMin, xMax, yMax |
   | 136 | 2 | `units_per_EM` |
   | 138 | 2 | `ascender`, signed |
   | 140 | 2 | `descender`, signed |
   | 142 | 2 | `height`, the line spacing |
   | 152 | 8 | `glyph`, the address of the glyph slot |
   | 160 | 8 | `size`, the address of the size record |
   | 168 | 8 | `charmap`, the selected character map |

8. **The glyph slot's fields are at these offsets**, and the outline
   and the bitmap sit inside the slot rather than behind a pointer.

   | Offset | Width | Field |
   | --- | --- | --- |
   | 24 | 4 | `glyph_index` |
   | 48 | 64 | `metrics`, eight 26.6 values: width, height, horiBearingX, horiBearingY, horiAdvance, vertBearingX, vertBearingY, vertAdvance |
   | 112 | 8 | `linearHoriAdvance`, the unrounded advance in 16.16 |
   | 128 | 16 | `advance`, the 26.6 x and y the pen moves |
   | 152 | 40 | `bitmap` |
   | 192 | 4 | `bitmap_left`, signed |
   | 196 | 4 | `bitmap_top`, signed |
   | 200 | 40 | `outline` |

   A bitmap is `rows` at 0, `width` at 4, `pitch` at 8, the address of
   the pixels at 16, `num_grays` at 24 and `pixel_mode` at 26, each
   four bytes wide but the last two. An outline is `n_contours` at 0
   and `n_points` at 2, both two bytes and signed, then the address of
   the points at 8, of the tags at 16 and of the contour end indices at
   24. A point is two eight-byte coordinates, and a tag is one byte
   whose lowest bit is 1 for a point on the curve.

9. **A size's metrics sit 24 bytes into the size record.** Within them
   `x_ppem` is at 0 and `y_ppem` at 2, both two bytes; `x_scale` at 8
   and `y_scale` at 16, both 16.16; and the scaled `ascender`,
   `descender`, `height` and `max_advance` at 24, 32, 40 and 48, all
   26.6.
10. **Scaled coordinates are 26.6 and scaling factors are 16.16.**
    Divide a 26.6 value by 64 for whole pixels. `FT_MulFix` is the
    call that applies a 16.16 factor to a design-unit value, and it
    rounds the way FreeType's own scaling does.
11. **`FT_Set_Transform` changes the outline and the advance, not the
    metrics.** The glyph slot's `metrics` are the untransformed
    glyph's whatever the matrix says; `advance` at offset 128 is
    transformed. Passing 0 for both arguments restores the identity.
12. **A string a face answers belongs to the face.** `family_name`,
    `style_name` and `FT_Get_Postscript_Name` stop being valid when
    the face is released. Copy each with `ptr.read_str` first.
13. **A face is reference counted.** `FT_Reference_Face` adds one, and
    the face needs as many `FT_Done_Face` calls as it has references.
14. **`FT_New_Memory_Face` does not copy the font.** The bytes must
    stay alive and unchanged until the face is released.
15. **A bitmap the caller owns is initialised before it is used.**
    `FT_Bitmap_Init` zeroes the 40-byte record, and without it the
    library reads uninitialised bytes as an existing buffer.
    `FT_Bitmap_Done` releases what it allocated. A bitmap that belongs
    to a glyph slot is the slot's and is never passed to either call.
16. **A name string in the `name` table is not text and is not
    null-terminated.** `FT_Get_Sfnt_Name` answers the address at
    offset 8 and the length at offset 16, and the platform and
    encoding identifiers at offsets 0 and 2 say how to read the bytes.
    Platform 3 with encoding 1 is UTF-16 big-endian.
17. **`FT_Load_Sfnt_Table` with a length slot holding 0 answers the
    length.** The slot decides what the call does, and a buffer passed
    beside a slot holding 0 is not written to. Call it twice: once to
    size the buffer, once to fill it. `FT_Sfnt_Table_Info` with a tag
    of 0 answers the number of tables the same way.
18. **The load flags are a bit set.** 0 is the default. 1 skips the
    scaling and answers design units, 2 skips the hinter, 4 renders in
    the same call and 8 ignores the bitmap strikes.
19. **The render modes are 0 for eight-bit grey, 1 for that same grey
    under the light hinting target, 2 for one bit per pixel, 3 and 4
    for the horizontal and vertical subpixel modes and 5 for the
    signed distance field.** The pixel mode the bitmap reports is 1
    for monochrome and 2 for grey.
20. **Glyph index 0 is the substitute glyph, not an error.**
    `FT_Get_Char_Index` and `FT_Get_Name_Index` answer 0 for something
    the font does not have, and loading glyph 0 succeeds.

## What is not included

- **`FT_Outline_Decompose`.** It takes an `FT_Outline_Funcs` structure
  whose four members are C function pointers, and the novo-lang
  foreign function interface passes integers, floats and strings. A
  program that wants the contours reads the point array, the tag array
  and the contour end indices out of the `FT_Outline` record by hand;
  rule 8 gives the offsets.
- **`FT_Outline_Render`.** `FT_Raster_Params` carries a `gray_spans`
  callback. `FT_Render_Glyph` rasterises into the glyph slot's own
  bitmap and needs none.
- **`FT_Open_Face`.** `FT_Open_Args` can name a custom `FT_Stream`,
  whose read and close members are function pointers. `FT_New_Face`
  and `FT_New_Memory_Face` cover a path and a block of memory.
- **`FT_New_Library` and `FT_Set_Debug_Hook`.** The first takes a
  structure of allocator function pointers and the second takes a
  function pointer. `FT_Init_FreeType` builds a library with the
  default allocator and the default modules.
- **The cache subsystem.** Every `FTC_` entry point hangs off
  `FTC_Manager_New`, which takes a face requester callback.
- **The colour glyph calls.** `FT_Get_Color_Glyph_Paint` and the
  COLRv1 calls beside it pass an `FT_OpaquePaint` by value.
- **The variable font calls.** `FT_Get_MM_Var` answers a structure
  whose axis and named-instance arrays are variable length. The whole
  family is left out of the first release.
- **The stroker.** The fifteen `FT_Stroker_` entry points turn an
  outline into its outlined border. They are left out of the first
  release.
- **`FT_Get_Advances`.** The array form answers 0 for every glyph
  under the no-scale flag on the TrueType driver, which is a different
  contract from the single-glyph call. `FT_Get_Advance` is here.
- **`FT_Has_PS_Glyph_Names`.** It answers 0 for a TrueType face whose
  `post` table carries names and which `FT_Get_Glyph_Name` reads
  successfully. Read `FT_Get_Glyph_Name`'s error code instead.

## Related packages

`font-nv` is the font port written in novo-lang, with no C library. It
covers reading the outlines and the metrics out of a font file. It does
not rasterise at FreeType's quality, and rasterising is what this
package is for. `font-nv` is planned and not published yet.

Choose `font-nv` when the program must build for a microcontroller or
for WebAssembly, when a C toolchain is not wanted, or when the job is
reading a font rather than drawing one. Choose this package when the
program has to put antialiased text on a screen and match what every
other program on the machine draws.

`libharfbuzz-sys` binds HarfBuzz, which decides which glyphs a run of
text uses and where they go. FreeType draws the glyphs HarfBuzz
chooses; the two are used together and neither replaces the other.

## Tests

`tests/libfreetype_tests.nv` holds fourteen tests over the fifty-four
entry points. They call the C library, so `novo test` needs FreeType
installed and linkable:

```
novo test tests/libfreetype_tests.nv
```

`novo pkg build` type-checks the declarations and needs nothing
installed.

The suite carries its own font as hex: a 776-byte TrueType font with
three glyphs, a square mapped from U+0041 and a triangle mapped from
U+0042 beside the substitute glyph. Thirteen of the fourteen tests load
it from memory and read and write nothing. The fourteenth writes a file
that is not a font under `/tmp`, to see which error a face reports for
it, and deletes it again.

The tests assert the face's own description against the font that was
built for them, the two directions of the character map, the glyph
names, the size metrics at three sizes, the glyph metrics and the
advance of the square, the control box and the exact bounding box of an
outline that has no curves, a copied outline moved and doubled in
width, the grey bitmap the square rasterises to, the four SFNT table
calls, and FreeType's own rounding in all six fixed-point calls.

`novo --leak-check` reports six leaked objects at the end of the run.
They are the `ptr.read_str` copies the tests make out of the face's own
strings; `ptr.read_str` is declared untracked, which is a defect in the
toolchain and not in this package.

## Licence

Apache-2.0. See [LICENSE](LICENSE).

FreeType itself is distributed under the FreeType Licence and the GPL
version 2, and installing it is the reader's own step.
