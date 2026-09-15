# i18n-nv

[Project Fluent](https://projectfluent.org/) is a localisation system
whose file format, FTL, lets a translator write a whole message,
including the grammar it needs, as one entry. This package implements
FTL for novo-lang: the parser, the resolver, BCP 47 locale negotiation,
the CLDR plural rules, and a reader for gettext `.po` catalogues. It is
built on [numfmt-nv](https://novo-lang.org/packages/numfmt-nv) for
numbers and [calendar-nv](https://novo-lang.org/packages/calendar-nv)
for dates.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What it is

**Localisation** is producing the text a program shows in the language
of the person reading it. A **message** is one such piece of text, kept
under an identifier the program uses. A **catalogue** or **resource**
is a file of them for one language.

**FTL** is Fluent's file format, described in the
[Fluent syntax guide](https://projectfluent.org/fluent/guide/). A
message is `hello = Hello, world`. A **placeable**, written in braces,
is a hole in the text: `{$name}` takes a value from the program,
`{message-id}` takes another message's text, and `{NUMBER($n)}` calls a
built-in function. An **attribute** is a secondary string under a
message, such as the accessible label of a button. A **term**, written
`-brand-name`, is a message meant only for other messages to reference.
A **selector** picks one of several variants, most often by the plural
category of a number.

A **locale** is identified by a BCP 47 tag such as `en`, `da` or
`pt-BR`. **Negotiation** is choosing which of the locales a program
ships comes closest to what the reader asked for. A **fallback chain**
is the ordered list to try, so a message missing from one locale is
looked for in the next.

A **plural category** is one of the six names CLDR gives the grammatical
forms a language counts in: `zero`, `one`, `two`, `few`, `many` and
`other`. English uses two of them. Japanese uses one. Arabic uses all
six. The category is not a function of the number alone: CLDR defines
six **operands** over a formatted number, and three of them describe
its digits rather than its value. `v` is the count of visible fraction
digits, `f` is their value, and `t` is that with trailing zeros
removed. In English `1` is `one` and `1.0` is `other`.

**Bidirectional isolation** is a pair of invisible characters, U+2068
FIRST STRONG ISOLATE and U+2069 POP DIRECTIONAL ISOLATE, placed around
an inserted value. They stop a Hebrew sentence containing an English
username from rendering with its punctuation in the wrong place.

## Install

```
novo pkg add i18n-nv
```

## Example

```novo
use ftlparse
use i18nbundle

fn main() [io]
    // An FTL source. The backslash escapes the dollar for novo's own
    // string interpolation; the FTL text itself reads `{$name}`.
    let source = "hello = Hello, {\$name}!\n"

    // Parse it. The pair's second half is every entry the parser could
    // not read; the resource holds everything it could.
    let parsed = ftlparse.parse(source)

    // A bundle is one locale's messages. Adding a resource answers a
    // new bundle rather than changing the one passed in.
    let empty = i18nbundle.bundle("en", i18nbundle.default_options("en"))
    let b = i18nbundle.with_resource(empty, parsed.0)

    // Resolving always produces text. `errors` lists what went wrong.
    let out = i18nbundle.format(b, "hello", [I18nArgStr("name", "Ada")])
    println(out.text)
    for e in out.errors
        println(e.message())
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: i18n-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `ftlparse` | FTL text parsed into messages, terms, attributes, patterns and selectors, with the entries it could not read recorded separately. |
| `i18nlocale` | BCP 47 tags, matching at three strictnesses, negotiation, `Accept-Language` parsing and the fallback chain. |
| `i18nplural` | The CLDR categories, the six operands they are computed from, and the shipped locales' rules as data. |
| `i18nfmt` | Number and date formatting for a locale, and the operands the plural selector needs, from one number format. |
| `i18nbundle` | A bundle of one locale's resources, a chain of bundles, and the resolution that produces text and errors. |
| `i18npo` | gettext `.po` catalogues read, their `Plural-Forms` expression evaluated, and what a conversion to FTL cannot carry. |

## How to choose an entry point

**`i18nbundle.format` resolves one message from one locale.**
`format_attribute` resolves one of its attributes.

**`i18nbundle.format_in` resolves against a chain.** An `I18nChain` is
the ordered list of bundles, so a message missing from the first locale
is looked for in the next. `answering_bundle` says which one answered.

**`format_into` appends to a buffer you own** and hands back the buffer
and the error list. A static site generator rendering many messages
into one page wants this.

**`ftlparse.parse` keeps what it could read; `parse_strict` refuses the
file.** They are two functions because the two callers are different
programs. A build step should fail over a broken translation. The
application reading it at run time must not.

**`i18nfmt.operands_of` is what connects formatting to
pluralisation.** It builds the CLDR operands from the same
`I18nNumFormat` the number will be printed with. See rule 3.

**`i18nlocale.negotiate` ranks a list of requests against a list of
available locales; `chain_for` answers the fallback chain for one
request.** `parse_accept_language` turns an HTTP header into the
request list.

## The rules a user needs

1. **A resolution always produces text.** `I18nResolved` is not a
   `Result`. A missing variable renders as `{$name}`, braces and all. A
   missing message renders its own identifier. A cyclic reference
   terminates and is reported. `errors` says what happened, so an
   application logs it, a test asserts on it and a build step refuses
   over it. `is_clean` is the one-line check.
2. **`I18nMissingMessage` and `I18nMissingEverywhere` mean different
   things.** One locale missing a message is a translation gap. Every
   locale in the chain missing it is a bug in the program.
3. **Format and pluralise with the same `I18nNumFormat`.** The plural
   category depends on the visible fraction digits, so a program that
   prints with one setting and selects with another says "1.0 file".
   `I18nBundleOptions` carries one number format for both, and
   `i18nfmt.operands_of` computes the operands from it. The CLDR
   [plural rules specification](https://unicode.org/reports/tr35/tr35-numbers.html#Language_Plural_Rules)
   defines the six operands.
4. **Ordinals are a second rule set.** English has two cardinal
   categories and four ordinal ones, for `1st`, `2nd`, `3rd` and `4th`.
   `I18nPluralKind` is therefore a parameter of `rules_for` rather than
   an assumption.
5. **A locale with no shipped rules gets English's.**
   `i18nplural.rule_is_fallback` reports it, and a resolution that used
   it reports `I18nPluralRulesMissing`. Without that report the gap is
   invisible for every count that happens to land in `other`.
6. **Bidirectional isolation is on by default.** Turn it off only in a
   test that compares strings, where the invisible marks make an
   assertion fail confusingly. `i18nbundle.test_options` is that
   setting. A user interface shipped with it has a defect that shows
   only in a right-to-left locale.
7. **Adding a resource answers a new bundle.** Nothing here mutates. A
   program holds a bundle as a value and swaps it atomically when a
   translation reloads.
8. **`i18nfmt.relative_into` takes both instants.** A formatter that
   read a clock would produce a different string on every run, and
   nothing here reads a clock.
9. **`q=0` in an `Accept-Language` header is a refusal, not a low
   ranking.** RFC 9110 section 12.5.4 says so, and
   `parse_accept_language` excludes such a tag rather than ranking it
   last. This is the rule a hand-written header parser gets wrong.
10. **`max_reference_depth` is 8.** A message reference chain deeper
    than that is treated as a cycle and reported. A legitimate FTL file
    is two or three deep.
11. **Check translations at build time with three published calls.**
    `ftlparse.variables_of` lists the variables a message references.
    `i18nbundle.missing_against` lists the identifiers a reference
    bundle has and this one does not. `i18nbundle.variable_drift` lists
    the messages whose variables differ from the reference's. A
    translation that references `$userName` where the source says
    `$username` is the commonest localisation defect, and nothing
    catches it at run time except a reader.
12. **A `.po` file's `msgstr[n]` index is a position, not a
    category.** Which CLDR category an index covers depends on the
    catalogue's own `Plural-Forms` expression. `i18npo.category_map`
    evaluates that expression over the counts CLDR distinguishes and
    reports what each index covers. The mapping can be ambiguous,
    because a `.po` file may partition differently, and the conversion
    says so rather than guessing.
13. **`PoLimit` names what a `.po` conversion cannot carry.** Fuzzy
    entries, untranslated entries, ambiguous categories, positional
    `%s` placeholders that need invented names, and folded contexts. A
    converter that answered only a resource would have dropped all five
    silently.
14. **Three FTL features have no gettext spelling.** Attributes,
    selectors on anything but a count, and terms.
    `i18npo.unsupported_features` is that list as data.
15. **Six locales ship at 0.1.0: `en`, `da`, `de`, `fr`, `es`, `ja`.**
    They cover the distinct shapes rather than the largest audiences.
    French puts 0 in the `one` category, which is the case a two-form
    design gets wrong, and Japanese makes no plural distinction at all.
    `i18nplural.shipped_locales` and `i18nfmt.shipped_locales` are
    published so a program can ask rather than assume.

## What is not included

- **Reading files.** FTL text and `.po` text arrive as strings the host
  read. Nothing here opens a path, consults the environment for a
  locale, or asks what time it is. All three are what make a localised
  string untestable when a library does them itself.
- **Printing.** `format_into` appends to a buffer the caller owns.
- **A clock.** `relative_into` takes the instant to measure from and
  the instant to measure against, both as calendar-nv values.
- **`i18npo`'s bodies at 0.1.0.** The interface is published now so its
  shape can be reviewed, and so a converter is not written outside the
  package in the meantime. The reader, the `Plural-Forms` evaluator and
  the category mapping are declared and will land after the Fluent
  half.
- **Locale data beyond the six shipped locales.** Forty locales' plural
  rules, number symbols and date symbols would be a data file rather
  than rows, and this package would then need the tiering
  [unicode-nv](https://novo-lang.org/packages/unicode-nv) has.
- **Running on a microcontroller.** The package makes no such claim and
  carries no device probe. A resource is a parse tree, a bundle is a
  list of them, and a resolution builds a string. Firmware with a
  display and several languages wants a compiled string table, which
  would be a different package.
- **Collation and locale-aware sorting.** A sort order is a much larger
  table than anything here.

## Related packages

- [numfmt-nv](https://novo-lang.org/packages/numfmt-nv) formats
  integers and floats. It supplies the digits, and its formatting
  settings are what the plural operands are computed from.
- [calendar-nv](https://novo-lang.org/packages/calendar-nv) is civil
  dates and times with no clock in them. `I18nArgDate` takes its
  `CivilDateTime` directly, so a caller holding a date passes it rather
  than converting.
- [unicode-nv](https://novo-lang.org/packages/unicode-nv) is the
  character database. This package does not depend on it. A program
  that needs case folding or normalisation over localised text adds it
  itself.
- [template-nv](https://novo-lang.org/packages/template-nv) substitutes
  values into a string with no grammar and no locale. Use it for a log
  line or a generated file. Use this package for text a person reads in
  their own language.

## Tests

The reference implementation is **Project Fluent**: the syntax guide
for `ftlparse`, and **`fluent-rs`** for the resolver's shape, including
`FluentBundle`, the error-collecting `format_pattern`, the isolation
setting, and the model where the application holds a list of bundles.
**CLDR** supplies the plural categories, the operand definitions and
the sample sets the tests are written from. **BCP 47** defines the
tags, and **RFC 9110 section 12.5.4** defines `Accept-Language`.

```bash
novo test tests/ftlparse_tests.nv      #  7 tests: messages, placeables, selectors, junk
novo test tests/i18nplural_tests.nv    # 11 tests: the categories and the operands
novo test tests/i18nbundle_tests.nv    #  8 tests: resolution, errors and the chain
novo test tests/i18npo_tests.nv        #  4 tests: the catalogue and its plural expression
```

The suite asserts that a file with one broken entry still yields every
other message, that a missing variable renders visibly and is reported,
that a cycle terminates, that `1` and `1.0` select different English
categories, that 0 is `one` in French, that Japanese has one category,
that a locale with no rules falls back and says so, that `q=0` excludes
a tag, and that an ambiguous `.po` index mapping is reported rather
than guessed.

The tests compile today and fail at run, each on the
`not implemented: i18n-nv.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

Nothing is implemented. The table lists the surface an implementation
has to fill.

| Item | Implemented |
| --- | --- |
| `ftlparse.parse`, `.parse_strict`, `.error_at`, `.write_into` | no |
| `ftlparse.message_of`, `.term_of`, `.attribute_of`, `.message_ids`, `.span_str` | no |
| `ftlparse.variables_of`, `.builtins` | no |
| `i18nlocale.parse`, `.canonical`, `.truncate_tag`, `.undetermined`, `.is_rtl` | no |
| `i18nlocale.matches`, `.negotiate`, `.chain_for`, `.parse_accept_language` | no |
| `i18nplural.category_of`, `.rules_for`, `.rule_is_fallback`, `.has_plural_forms` | no |
| `i18nplural.operands_of`, `.operands_of_int`, `.explicit_match` | no |
| `i18nplural.category_name`, `.category_named`, `.shipped_locales` | no |
| `i18nfmt.en_symbols`, `.symbols_for`, `.date_symbols_for`, `.shipped_locales` | no |
| `i18nfmt.integer_format`, `.decimal_format`, `.operands_of`, `.operands_of_int` | no |
| `i18nfmt.int_into`, `.float_into`, `.int_radix_into` | no |
| `i18nfmt.datetime_into`, `.date_into`, `.relative_into`, `.weekday_name`, `.month_name` | no |
| `i18nbundle.default_options`, `.test_options`, `.bundle`, `.with_resource`, `.chain` | no |
| `i18nbundle.format`, `.format_attribute`, `.format_in`, `.format_attribute_in` | no |
| `i18nbundle.format_into`, `.format_into_in`, `.is_clean` | no |
| `i18nbundle.has_message`, `.message_ids`, `.answering_bundle` | no |
| `i18nbundle.missing_against`, `.variable_drift`, `I18nError.message` | no |
| `i18npo.read_catalogue`, `.error_line`, `.header_of`, `.entry_of` | no |
| `i18npo.parse_plural_forms`, `.eval_plural`, `.category_map` | no |
| `i18npo.to_resource`, `.to_ftl_into`, `.unsupported_features` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
