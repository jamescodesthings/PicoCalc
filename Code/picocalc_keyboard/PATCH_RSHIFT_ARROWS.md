# Patch: RSHIFT + LEFT/RIGHT → HOME / END

## What this patch does

Adds `.symb` (shifted variant) entries for the LEFT and RIGHT D-pad
buttons in `btn_entries`, mirroring the existing UP/DOWN behaviour:

| Combo            | Stock firmware emits | After patch emits |
|------------------|----------------------|-------------------|
| RSHIFT + UP      | `KEY_PAGE_UP` (0xD6) | unchanged         |
| RSHIFT + DOWN    | `KEY_PAGE_DOWN`(0xD7)| unchanged         |
| RSHIFT + LEFT    | **nothing** (silent) | `KEY_HOME` (0xD2) |
| RSHIFT + RIGHT   | **nothing** (silent) | `KEY_END`  (0xD5) |

## Why the stock firmware blackholes RSHIFT+LEFT/RIGHT

`keyboard.ino`, the `transition_to()` function (around line 145):

```c
if (shift && (chr <'A' || chr >'Z')) {
  chr = p_entry->symb;
}
```

When EITHER shift is held, `chr` is unconditionally replaced with
`p_entry->symb`. The `btn_entries` array originally only filled `.symb`
for UP and DOWN (PgUp / PgDn). The LEFT and RIGHT entries were
single-element initialisers, so `.symb` defaulted to `0`.

Then, around line 194:

```c
if (chr != 0 && output==true) { ... }
```

…the zero `chr` is filtered out, so the firmware never emits anything
for RSHIFT+LEFT or RSHIFT+RIGHT.

This patch fills the missing `.symb` slots so the same shift-replace
mechanism produces `KEY_HOME` and `KEY_END` instead of nothing.

## Diff

```diff
@@ static const struct entry btn_entries[NUM_OF_BTNS] =
   {']','}'},
   {'[','{'},
-  {KEY_RIGHT},
+  {KEY_RIGHT,KEY_END},
   {KEY_UP,KEY_PAGE_UP},
   {KEY_DOWN,KEY_PAGE_DOWN},
-  {KEY_LEFT}
+  {KEY_LEFT,KEY_HOME}
 };
```

## Verification

After flashing, capture I2C scancodes with the
[`picocalc_trixie`](https://github.com/jamescodesthings/picocalc_trixie)
`keyboard/debug` script. Expected sequences:

- RSHIFT held + LEFT  → `Scancode 210` (0xD2 = KEY_HOME)
- RSHIFT held + RIGHT → `Scancode 213` (0xD5 = KEY_END)

The Linux kernel driver in `picocalc_trixie` already maps these
scancodes through `rshift_macros`, so no driver-side change is needed
once this firmware is flashed.

## Filed upstream

Open PR for `clockworkpi/PicoCalc` — see commit history.
