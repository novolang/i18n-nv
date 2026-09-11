# i18n-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

Project Fluent for novo-lang, sans-IO: FTL text in, a localised string
out, and a list of everything that was wrong with it.

- `ftlparse` — FTL as a typed value: messages, attributes, placeables,
  selectors, and the junk it kept;
- `i18nlocale` — BCP 47 tags, negotiation, and the fallback chain;
- `i18nplural` — the CLDR categories, and the operands they are
  actually a function of;
- `i18nfmt` — numbers and dates delegated to numfmt-nv and calendar-nv,
  and the bridge that makes the selector agree with the formatter;
- `i18nbundle` — `I18nResolved`: the text, and the errors;
- `i18npo` — gettext, and exactly how far it maps.

```
novo pkg add i18n-nv
novo pkg build
novo test
```

## The one example that will work

```novo ignore
use ftlparse
use i18nbundle

// One locale's messages, and one message resolved.
fn greet(ftl: Str, name: Str) -> Str
    let parsed = ftlparse.parse(ftl)
    let b = i18nbundle.with_resource(
        i18nbundle.bundle("en", i18nbundle.default_options("en")), parsed.0)
    i18nbundle.format(b, "hello", [I18nArgStr("name", name)]).text
```

## The load-bearing interface: `I18nResolved`

```novo ignore
pub struct I18nResolved
    text: Str             // ALWAYS something a person can be shown
    errors: [I18nError]
```

**Not a `Result`.** A localisation defect is a *content* defect: a
translator referenced `$userName` where the source says `$username`, a
message id was renamed and one of nine files was missed, a new locale
has no plural rules. Every one of those ships at three in the morning,
in a locale nobody on the team reads, and the correct behaviour is
always the same — render the best text available and report what went
wrong.

An API that answered `Result` makes every call site choose between two
wrong things. `unwrap` is a crash in Japanese and nowhere else, which
is how a localisation bug becomes an outage. Writing a fallback at the
call site means writing one at every call site, and the fallbacks then
disagree — some show the id, some show English, some show nothing — so
the interface is inconsistent in exactly the locale nobody is looking
at.

So: a missing variable renders as `{$name}`, braces and all, which is
ugly and visible and points at the bug. A missing message renders its
own id. A cyclic reference terminates and is reported. Every one of
them produces text, and the error list says which happened — so an
application logs it, a test asserts on it, and a build step refuses
over it. `fluent-rs` reached the same design, and `format_pattern`'s
error-collecting signature is the reason it is usable in a browser.

The same principle runs one level down: `ftlparse.parse` keeps every
message it could read and records the junk, so an FTL file with one
broken entry localises everything else.

## The second decision: a plural category is not a function of a number

CLDR defines six operands, and three of them are about the digits
rather than the value — `v` is the **count of visible fraction
digits**, `f` is their value, `t` is that with trailing zeros removed.
In English `1` is `one` and `1.0` is `other`. Same value, different
category.

A progress indicator formatted to one decimal place says "1.0 files"
and not "1.0 file", and an API shaped `plural(n: Float)` could not know
which. So `i18nplural.category_of` takes `I18nOperands`, and
`i18nfmt.operands_of` builds one **from the same `I18nNumFormat` the
formatter will use**. That is why `I18nBundleOptions` carries one
number format rather than leaving formatting and pluralisation to be
configured separately: a program that formats with one setting and
pluralises with another says "1.0 file", and nothing in either library
could have caught it.

Ordinals are a second rule set for the same locale — `1st`, `2nd`,
`3rd`, `4th` are four categories in English where cardinals have two —
so `I18nPluralKind` is a parameter and not an assumption.

## The rules are data, and here is how a locale is added

`I18nPluralRule` is a value, the shipped locales' rules are rows, and
`rules_for` is a lookup. Adding a locale is:

1. a row in `i18nplural`'s table: the canonical tag, the kind, the
   categories the locale uses in CLDR's order with `I18nOther` last;
2. a row in `i18nfmt`'s `I18nNumSymbols` and `I18nDateSymbols` tables,
   if the locale's numbers or dates differ from English's;
3. the locale's name added to `shipped_locales`, which is published so
   this README cannot drift from the code;
4. a test case per category, from CLDR's own `plurals.xml` sample set.

No branch anywhere grows, and nothing else recompiles.

**Shipped at 0.1.0: `en`, `da`, `de`, `fr`, `es`, `ja`** — six locales
covering the distinct shapes rather than six chosen for reach: four
with English's two categories, `fr` where **0 is `one`** (the case a
two-form design gets wrong), and `ja` with no plural distinction at all.
Two more are worth having early and are named rather than promised:
`pl`, whose `few` and `many` break any design that assumed two forms,
and `ar`, which uses all six.

A locale with no rules gets English's, `rule_is_fallback` says so, and
`I18nPluralRulesMissing` reports it at resolution — because that gap is
otherwise invisible for every count that happens to land in `other`.

## Bidi isolation is on by default

Every placeable is wrapped in U+2068 FIRST STRONG ISOLATE and U+2069
POP DIRECTIONAL ISOLATE. They are invisible, and they are what stops a
Hebrew sentence with an English username in it from rendering with its
punctuation in the wrong place.

`I18nBundleOptions.use_isolating` exists only so a test that compares
strings can turn it off — the marks make every such assertion fail
confusingly. `i18nbundle.test_options` is that, and its doc comment
says shipping a UI with it is a bug, and one that shows only in a
right-to-left locale.

## The build-step checks are in the package

The commonest localisation defect is a translation that references
`$userName` where the source references `$username`: it produces a
message with a hole in it, for exactly the locale nobody on the team
reads, and nothing catches it at run time except a user.

So `ftlparse.variables_of`, `i18nbundle.missing_against` and
`i18nbundle.variable_drift` are published, and `ftlparse.parse_strict`
refuses a file that `parse` would have salvaged. Two calls rather than
a flag, because the two callers are different programs with different
correct behaviour: a build should fail over a broken translation, and
the application that reads it at run time must not.

## What gettext `.po` compatibility would take

`i18npo` is declared with its reader, and **its bodies are not shipped
at 0.1.0** — the interface is published now so the shape is reviewed
while it is cheap, and so nobody writes the converter outside the
package in the meantime. Four things, and only the first is
mechanical:

1. **The file format.** `msgid`, `msgstr`, `msgctxt`, `msgid_plural`,
   `msgstr[n]`, the comment kinds, the C escapes, the continuations.
   Tedious and finite: `read_catalogue`.

2. **The `Plural-Forms` header is a C expression.**
   `plural=(n%10==1 && n%100!=11 ? 0 : …)` is a program, in a small
   language with `%`, the comparisons, `&&`, `||` and the ternary.
   Reading a `.po` file means writing an evaluator for it — which is
   `PoPluralExpr` and `eval_plural`, declared rather than pretended
   away.

3. **The indices are not categories.** `msgstr[0]` is a *position*, and
   which CLDR category it corresponds to depends on that expression.
   `category_map` evaluates it over the counts CLDR distinguishes and
   reports what each index covers — and the mapping can be **ambiguous**,
   because a `.po` file is free to partition differently. When it is,
   the conversion says so rather than guessing.

4. **Three Fluent features have no `.po` spelling**, and they are the
   three that make Fluent worth having: attributes, selectors on
   anything but a count, and terms. `unsupported_features()` is that
   list as data, so a migration tool prints the sentence this package
   would.

`PoLimit` names the five things a conversion cannot carry — fuzzy
entries, untranslated entries, ambiguous categories, positional `%s`
placeholders that have to be given invented names, and folded contexts
— because a converter that answered only a resource would have dropped
all five silently.

## The layer, and why

`core`. FTL text arrives as a `Str` the host read; a bundle holds
parsed resources; the instant a relative-time formatter measures
against is a `CivilDateTime` the caller passes in. Nothing here reads a
file, consults the environment for a locale, or asks what time it is —
all three are the host's, and all three are what makes a localised
string untestable when a library does them itself.

`i18nfmt.relative_into` takes **both** instants for exactly that
reason: a formatter that called a clock would produce a different
string on every run.

Adding a resource answers a **new** bundle rather than mutating one,
because `[mutate]` is a host effect and this package has none — and
because a bundle is then a value a program can hold and swap atomically
when a translation reloads.

## `@tier(embedded)` is not claimed

There is no device consumer. A resource is a parse tree, a bundle is a
list of them, a resolution builds a string, and the error list is a
list. A firmware with six fixed strings does not want a message format
at all; one with a display and several languages wants a compiled
string table, which is a different package rather than an annotation on
this one.

## The reference implementation

Project Fluent: the [syntax
guide](https://projectfluent.org/fluent/guide/) for `ftlparse` and
`fluent-rs` for the resolver's shape — `FluentBundle`, the
error-collecting `format_pattern`, `set_use_isolating`, and the
application-holds-a-list-of-bundles fallback model. CLDR's
`plurals.xml` for the categories, their sample sets for the tests, and
the plural-rules specification for the operands. BCP 47 for the tags
and RFC 9110 § 12.5.4 for `Accept-Language` — where `q=0` is a
**refusal**, so a locale carrying one is excluded rather than ranked
last, which is the rule a hand-rolled header parser gets wrong.

## Dependencies

**`numfmt-nv ^0.1.1`** for the digits, and for the operands: the plural
category and the printed number have to come from one formatting, and
`i18nfmt.operands_of` is what makes them.

**`calendar-nv ^0.0.2`** for `CivilDate` and `CivilDateTime`, so a
caller that already holds a date passes it rather than converting, and
because the day and month arithmetic a relative-time formatter needs is
already there and correct. No clock: calendar-nv has none either.

Both are `core`, which is what keeps this package `core`.

## The consumers, and what adopting this would take

**`orbit/website` and the registry's package pages** are the first
consumer with a real need: the site is a staged launch and the
landing page is a holding page today, but every string on it is
English in a template, and a second language means either a second
template or this. `format_into` over the page's own buffer is the shape
a static generator wants.

**`std.cli`** is the second, and it is the one that would show the
design working: a command-line tool's `--help` output, its error
messages and its confirmation prompts are exactly the corpus Fluent is
for, and `2 files deleted` against `1 file deleted` is the case the
plural rules exist for. It also names the limit — `std.cli` is a
standard-library module and cannot depend on an orbit package, so the
adoption is a `novoterm`- or `novim`-shaped consumer first, and a
graduation later.

**novim** and **novoterm** are plausible eventual consumers and are not
consumers now: both are English-only, and both would want the
`i18nfmt` half (a file size, a line count) before the `ftlparse` half.

**A build step** is the consumer that pays for itself immediately, and
it needs no application at all: `parse_strict` plus `missing_against`
plus `variable_drift` over a translation directory is a check a
repository can run from the day the first second language lands.

## What a row wanted to widen

Nothing. Every function here is `[]`.

Three things the plan's row ("message catalogues and plural rules") did
not anticipate, recorded as the redesign input:

**The plural rules are not the hard part; the operands are.** The row
reads as though a plural rule is a table lookup. It is, once you have
the operands — and the operands are a function of the formatting, which
means the formatter and the selector are one design and not two. That
is why `i18nfmt` exists at all rather than the number placeable calling
numfmt-nv directly.

**The locale data is bigger than the code, and it is not shipped
whole.** Six locales' plural rules, number symbols and date symbols are
rows; forty locales' would be a data file, and this package would then
need the same tiering unicode-nv and slug-nv both needed — a compiled
tier and a blob a host reads. The interface does not have that split
yet, and the number to watch is the size of the tables at 0.1.0. Adding
it later costs an `I18nLocaleData` parameter on `rules_for`,
`symbols_for` and `date_symbols_for`, which is a breaking change; the
implementation lane should decide before the bodies land rather than
after.

**`i18npo`'s bodies are not scheduled.** The interface is published
because the shape is worth reviewing and because a converter written
outside the package would be written once per project. Naming that
here, rather than leaving it as a module that happens to be empty, is
the honest form.

## The surface

| module | `pub fn` | `pub struct` | `pub enum` |
| --- | --- | --- | --- |
| `ftlparse` | 11 | 8 | 3 |
| `i18nlocale` | 9 | 1 | 1 |
| `i18nplural` | 10 | 2 | 2 |
| `i18nfmt` | 16 | 3 | 2 |
| `i18nbundle` | 17 | 4 | 2 |
| `i18npo` | 10 | 2 | 3 |
| **total** | **73** | **20** | **13** |
