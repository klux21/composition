# Changelog of Data Composition Format Specification

## data_composition.1.0.1 / 2026-06-29

 - Earlier versions allowed entry names followed by a blocks without a mandatory `=` that
   makes the entry name the name of the following block. This caused ambiguity between
   intentionally unnamed blocks and blocks who were intended to represent an entries value.
   To resolve this, the opening curly brace of a block `{` must be preceded by a `=` now for
   becoming the value of an entry. If not preceded by a `=` blocks are still valid entries
   but the name of such a block is empty now.

## data_composition.1.0.0 / 2026-06-28

 - Initial version of the data composition format standard.
