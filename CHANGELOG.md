# Changelog

All notable changes to i18n-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `ftlparse` — FTL as a typed value with spans into the caller's
  source, and two entry points: `parse` keeps everything it could read,
  `parse_strict` refuses the file, because a build and an application
  have different correct behaviour.
- `i18nlocale` — BCP 47 tags parsed rather than split, three matching
  policies, and a negotiated chain whose last element is always the
  default.
- `i18nplural` — the six CLDR categories over `I18nOperands`, cardinal
  and ordinal rule sets, and the shipped locales as data.
- `i18nfmt` — separators and date symbols as rows, numbers and dates
  delegated to numfmt-nv and calendar-nv, and `operands_of` as the
  bridge.
- `i18nbundle` — `I18nResolved`, bundles, chains, and the two
  build-step comparisons.
- `i18npo` — the gettext reader, its plural-expression evaluator, and
  the five limits a conversion reports.

### Known

- **`I18nResolved` is the load-bearing interface, and resolution never
  fails.** A `Result` API makes every call site choose between a crash
  in Japanese and a fallback written nine times.
- **A plural category is a function of the FORMATTED number**, not of
  the number: `1` is `one` and `1.0` is `other`, so the formatter and
  the selector share one `I18nNumFormat`.
- **The rules are data.** Adding a locale is a row and a test; the
  README says exactly what a row is, and `shipped_locales` is published
  so the README cannot drift from it.
- **Bidi isolation is on by default**, and `test_options` turns it off
  for tests alone.
- **The build-step checks are in the package** — `variable_drift` is
  the one that catches the commonest defect there is.
- **`i18npo`'s bodies are not scheduled.** Its interface is published
  so the shape is reviewed and so nobody writes the converter outside
  the package.
- **`@tier(embedded)` is not claimed**; a firmware with a display wants
  a compiled string table, which is a different package.
- **Two dependencies**, numfmt-nv and calendar-nv, both `core`. No
  clock anywhere: a relative time takes both instants.

### Design notes

Adding a locale is four rows and no new branch: a row in `i18nplural`'s
rule table with the categories in CLDR's order and `I18nOther` last; a
row in `i18nfmt`'s number and date symbol tables if they differ from
English's; the tag added to `shipped_locales`; and a test case per
category from CLDR's own sample sets. `pl`, whose `few` and `many`
break any two-form design, and `ar`, which uses all six categories, are
the two worth adding early.

The locale data is bigger than the code and is not shipped whole. Six
locales are rows; forty would be a data file, and the package would
then need the tiering unicode-nv has, which costs an `I18nLocaleData`
parameter on `rules_for`, `symbols_for` and `date_symbols_for`. That is
a breaking change, so the implementation lane should decide before the
bodies land rather than after.

The consumers the surface was designed against: the project website and
the registry's package pages, whose strings are English in templates
today and which want `format_into` over the page buffer; a build step
over a translation directory, which needs no application at all and is
`parse_strict` plus `missing_against` plus `variable_drift`; and
`std.cli`, whose help text, errors and prompts are the corpus Fluent is
for, though a standard-library module cannot depend on an Orbit package
so an application-shaped consumer comes first.

`i18npo`'s bodies are not scheduled for 0.1.0. The interface is
published because the shape is worth reviewing and because a converter
written outside the package would be written once per project.
