# Unit Naming

This document describes how Victoria II parses country-specific unit names and selects names for newly created units.

## Country `unit_names` data

Country files under `common/countries` can define a `unit_names` block with lists grouped by unit type. In the vanilla game, these lists are used to provide names for ships. For example:

```txt
unit_names = {
    manowar = {
        "Name 1"
        "Name 2"
    }
}
```

The parser function at VA `0x004FBC10` handles the `unit_names` token (`0x576`). It looks up each child key in the subunit-type registry and stores the corresponding names in country data, indexed by subunit type.

## Selecting a unit name

The main name-selection routine is at VA `0x004FA8B0` (RVA `0x000FA8B0`). It uses the unit type to select that type's country-specific name list, checks candidate names against names already in use, and updates the list cursor as names are considered. If no suitable list name is available, the caller can use the normal generated-name path.

A field at offset `+0x39` in the unit-type data controls whether the routine scans the country-specific list. In the analyzed vanilla executable, ship types take the list-scanning path, while land-unit types skip it and proceed to generated names such as `1st New York Infantry`.

The country data contains per-type list and cursor information around offsets `+0x994` and `+0x9A4`, respectively. The unit type's ID at offset `+0x24` is used to select the corresponding entries.
