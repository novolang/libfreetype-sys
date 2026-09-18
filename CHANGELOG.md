# Changelog

All notable changes to libfreetype-sys are recorded here. The format
is [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.1 — 2026-09-18

The documentation and comments in plain prose; no declaration changed.

## 0.1.0 — 2026-09-16

The first release: fifty-four entry points of the FreeType C API, one
`@ffi` declaration each, and no logic.

### Added

- `libfreetype` — the whole surface, in nine groups.
  - The library: `FT_Init_FreeType`, `FT_Done_FreeType`,
    `FT_Library_Version`, `FT_Error_String` and `FT_Property_Set`.
  - The face: `FT_New_Face`, `FT_New_Memory_Face`, `FT_Done_Face`,
    `FT_Reference_Face` and `FT_Get_Postscript_Name`.
  - Character maps and glyph names: `FT_Select_Charmap`,
    `FT_Set_Charmap`, `FT_Get_Char_Index`, `FT_Get_Name_Index`,
    `FT_Get_Glyph_Name`, `FT_Get_First_Char` and `FT_Get_Next_Char`.
  - Sizes and the transform: `FT_Set_Char_Size`, `FT_Set_Pixel_Sizes`,
    `FT_Request_Size`, `FT_Select_Size` and `FT_Set_Transform`.
  - Loading and rendering: `FT_Load_Glyph`, `FT_Load_Char`,
    `FT_Render_Glyph`, `FT_Get_Kerning` and `FT_Get_Advance`.
  - Outlines: `FT_Outline_New`, `FT_Outline_Done`, `FT_Outline_Copy`,
    `FT_Outline_Translate`, `FT_Outline_Transform`,
    `FT_Outline_Embolden`, `FT_Outline_EmboldenXY`,
    `FT_Outline_Reverse`, `FT_Outline_Get_CBox`,
    `FT_Outline_Get_BBox`, `FT_Outline_Get_Orientation` and
    `FT_Outline_Check`.
  - Bitmaps: `FT_Bitmap_Init`, `FT_Bitmap_Copy`, `FT_Bitmap_Convert`,
    `FT_Bitmap_Embolden` and `FT_Bitmap_Done`.
  - The SFNT tables: `FT_Load_Sfnt_Table`, `FT_Sfnt_Table_Info`,
    `FT_Get_Sfnt_Name_Count` and `FT_Get_Sfnt_Name`.
  - Fixed-point arithmetic: `FT_MulFix`, `FT_DivFix`, `FT_MulDiv`,
    `FT_RoundFix`, `FT_CeilFix` and `FT_FloorFix`.
- `tests/libfreetype_tests.nv` — fourteen tests over the entry points.
  They call the C library, so they need FreeType installed. The suite
  carries its own 776-byte TrueType font as hex and loads it from
  memory, so thirteen of the fourteen read and write nothing.

### The answers come out of the structures

FreeType answers almost nothing through a return value. A face's glyph
count, a glyph's metrics, the rendered bitmap and the outline are all
fields of a record the library owns, and the handle is the address of
that record. This release promises the offsets of those fields. They
are in the README's "The rules a user needs" section, and they are the
offsets of the `x86_64` System V layout that FreeType 2.13 compiles
to.

The standard library reads eight bytes at a time. A field narrower
than that is masked out of the word that contains it, and a signed one
is sign-extended by hand. That is the cost of the arrangement, and the
README's rule 4 is the recipe.

### Where the effects fall

A face keeps its font file open for its whole life and reads glyph and
table data out of it on demand, so the sixteen entry points that pull
data through a face declare `[io, ffi]`. The other thirty-eight touch
only memory the caller already holds and declare `[ffi]` alone, which
is why `FT_New_Memory_Face` is an `[ffi]` call where `FT_New_Face` is
not.

### Named as missing

**`FT_Outline_Decompose`, and the whole callback half of FreeType.**
It takes an `FT_Outline_Funcs` structure whose four members — move to,
line to, conic to and cubic to — are C function pointers, and a
novo-lang program cannot produce one. This is the call a program would
normally use to walk a glyph's contours. A caller reads the points out
of the `FT_Outline` record by hand instead: the point count, the tag
array and the contour end indices are at fixed offsets, and the
README's rule 8 gives them.

**`FT_Outline_Render` and the raster parameters.** `FT_Raster_Params`
carries a `gray_spans` callback, so direct rasterisation into a
caller's own span sink is out. `FT_Render_Glyph` rasterises into the
glyph slot's bitmap and needs no callback.

**`FT_Open_Face`.** `FT_Open_Args` can name a custom `FT_Stream`, whose
read and close members are function pointers. The two ordinary cases
have their own entry points: `FT_New_Face` for a path and
`FT_New_Memory_Face` for bytes already in memory.

**`FT_New_Library` and `FT_Set_Debug_Hook`.** `FT_New_Library` takes
an `FT_Memory`, a structure of allocator function pointers, and
`FT_Set_Debug_Hook` takes a function pointer outright.
`FT_Init_FreeType` builds a library with the default allocator and the
default modules.

**The cache subsystem.** `FTC_Manager_New` takes a face requester
callback, so the whole `FTC_` family is out.

**The colour glyph calls.** `FT_Get_Color_Glyph_Paint` and the COLRv1
calls beside it pass an `FT_OpaquePaint` by value, and the novo-lang
foreign function interface passes integers, floats and strings.

**The variable font calls.** `FT_Get_MM_Var` answers an `FT_MM_Var`
whose axis and named-instance arrays are variable length, and
`FT_Done_MM_Var`, `FT_Set_Var_Design_Coordinates` and
`FT_Get_Var_Design_Coordinates` go with it. The whole family is left
out of the first release.

**The stroker.** `FT_Stroker_New` and the fourteen calls around it turn
an outline into its outlined border. They are bindable and are left out
of the first release, because stroking is a different job from loading
and measuring a glyph.

**`FT_Get_Advances`.** The array form of `FT_Get_Advance` answers 0 for
every glyph under the no-scale flag on the TrueType driver, which is a
different contract from the single-glyph call. `FT_Get_Advance` is
here.

**`FT_Has_PS_Glyph_Names`.** It answers 0 for a TrueType face whose
`post` table carries names and which `FT_Get_Glyph_Name` reads
successfully, so the answer does not mean what the name says. Ask
`FT_Get_Glyph_Name` and read its error code instead.
