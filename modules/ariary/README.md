# ariary: Malagasy Ariary amounts

Formats, reads and converts amounts in Malagasy Ariary (ISO 4217 `MGA`), writes them in words,
and converts old Malagasy francs (FMG).

## Public API

Every language port provides each function below, named the way its language does it. Behaviour
is set by the rules and test vectors here. Parsers (text typed by users → value) come in three forms:
`parse` throws, `safeParse` returns the value or the error code, `tryParse` returns the value or
nothing.

| Function          | Does                                            | JavaScript                                    | PHP                                                                     |
| ----------------- | ----------------------------------------------- | --------------------------------------------- | ----------------------------------------------------------------------- |
| `formatAriary`    | Amount → `Ar 1 500` (grouping, symbol position) | `formatAriary(amount, options)`               | `Ariary::fromInt($amount)->format(...)`, `Ariary::fromText($text)`      |
| `parseAriary`     | Text → amount, or throws                        | `parseAriary(text, options)`                  | `Ariary::parse($text, ...)->amount()`, `->digits`                       |
| `safeParseAriary` | Text → amount or error code, never throws       | `safeParseAriary(text, options)`              | `Ariary::safeParse($text, ...)`                                         |
| `tryParseAriary`  | Text → amount, or nothing, never throws         | `tryParseAriary(text, options)` → `undefined` | `Ariary::tryParse($text, ...)` → `null`                                 |
| `spellAriary`     | Amount → Malagasy words + `ariary`              | `spellAriary(amount, options)`                | `Ariary::fromInt($amount)->spell(...)`, `Ariary::spellText($text, ...)` |
| `fmgToAriary`     | Old francs → Ariary (5 FMG = 1 Ar)              | `fmgToAriary(francs)`                         | `Ariary::fromFmg($francs)->amount()`, `->digits`                        |
| `ariaryToFmg`     | Ariary → old francs                             | `ariaryToFmg(amount)`                         | `Ariary::fromInt($amount)->toFmg()`, `->toFmgDigits()`                  |
| `formatFmg`       | Francs → `Fmg 7 500`                            | `formatFmg(francs, options)`                  | `Ariary::formatFmg($francs, ...)`                                       |

## Rules

1. **Amounts are numbers from 0 up, whole or decimal**, with any number of decimal places: the
   same range as the words module. The spec sets no maximum; a port stops at its largest exact
   integer: `Number.MAX_SAFE_INTEGER` (9 007 199 254 740 991) in JavaScript, `PHP_INT_MAX` in PHP
   for whole amounts and 9 007 199 254 740 991 for decimal ones. Negative amounts and amounts
   above that limit are `OUT_OF_RANGE`; anything that is not a finite number is `INVALID_TYPE`.
   Shared vectors stay within 9 007 199 254 740 991. A decimal amount is the number value in its
   shortest exact decimal form: `1500.50` is `1500.5`. **Every function that takes an amount
   also takes decimal text** (words rule 13: digits, an optional point and decimals, at most
   100 characters), read exactly and with no upper limit: `formatAriary('1500.25')` →
   `Ar 1 500,25`.
2. **Default format: `Ar 1 500`.** Symbol `Ar`, a no-break space (U+00A0), then the digits in
   groups of three separated by no-break spaces. **Decimals follow a comma**, as written, never
   grouped, rounded or padded: `1500.5` → `Ar 1 500,5`, `0.25` → `Ar 0,25`.
3. **Options** (`formatAriary` and `formatFmg`):

   | Option            | Values                            | Example for 1500.5 |
   | ----------------- | --------------------------------- | ------------------ |
   | `grouping`        | `space` (default), `dot`, `comma` | `Ar 1.500,5`       |
   | `decimal`         | `comma` (default), `dot`          | `Ar 1 500.5`       |
   | `symbolPosition`  | `before` (default), `after`       | `1 500,5 Ar`       |
   | `currencyDisplay` | `symbol` (default), `code`        | `MGA 1 500,5`      |

   The defaults are the Malagasy way. With `dot` grouping, decimals still follow a comma:
   `1234567.89` → `Ar 1.234.567,89`. **A comma is never a thousand separator in Madagascar**:
   `grouping: 'comma'` and `decimal: 'dot'` exist only for text in another convention
   (`Ar 1,500.5`). `grouping` and `decimal` must differ. `currencyDisplay: 'code'` writes the
   ISO 4217 code instead of the symbol: `MGA` for ariary, `MGF` for old francs. Any other value, or the same character
   for both, is `INVALID_TYPE`.

4. **Reading text** (`parseAriary`, `safeParseAriary`, `tryParseAriary`), with the `grouping` and
   `decimal` options of rule 3 (no `symbolPosition`):
   - Accepts the currency as `Ar`, `MGA` or `Ariary` (any case), before or after the amount,
     or no currency at all, but not on both sides.
   - Without `grouping`, digit groups may be separated by spaces, no-break spaces, narrow
     no-break spaces (U+202F) or dots, the same separator throughout. With `grouping`, only that
     separator (`space` takes all three spaces). Every group after the first has three digits.
   - Decimals follow one `decimal` separator, a comma by default: `1 500,50 Ar` → `1500.5`,
     `0,5` → `0.5`. **By default `1,500` is 1.5**, the Malagasy reading, not fifteen hundred
     (see "Recorded decisions"). For text in another convention, say so:
     `parseAriary('1,500.50', { grouping: 'comma', decimal: 'dot' })` → `1500.5`.
   - Rejected with `INVALID_FORMAT`: a decimal point other than the `decimal` option, a second
     decimal separator, a decimal separator with no digit on either side (`1,`, `,5`), signs,
     leading zeros, other currencies, and input longer than 329 characters
     (`limits.maxReadLength`: the longest text `formatAriary` writes for a number, so every
     formatted amount reads back).
   - **Never a rounded value.** An amount a number value cannot hold exactly (too many decimals,
     or above the port's largest exact integer) is `PRECISION_LOSS`. The option
     `{ as: 'text' }` gives the exact decimal digits instead: `parseAriary('0,123456789012345678',
{ as: 'text' })` → `'0.123456789012345678'`.
5. **In words:** the number in words (see the words module) followed by `ariary`:
   `1500` → `dimanjato sy arivo ariary`, `1500.5` → `dimanjato sy arivo faingo dimy ariary`.
   The words `system` option applies (words rule 12): `30 000 000` in the traditional system →
   `telo safatsiroa ariary`. The amount may also be decimal text (words rule 13), read exactly,
   for amounts too long for a number value: `'2000000000000000.9'` →
   `roa tapitrisa lavitrisa faingo sivy ariary`. Text that is not plain decimal is
   `INVALID_FORMAT`.
6. **Malagasy franc:** 1 Ar = 5 FMG. `fmgToAriary` divides by 5 and `ariaryToFmg` multiplies by
   5, both exactly in decimal: `7501` FMG → `1500.2` Ar, `0.07` Ar → `0.35` FMG (never a
   binary rounding artefact such as `0.35000000000000003`). Francs may be decimal too. A result
   a number value cannot hold exactly is `PRECISION_LOSS`; `{ as: 'text' }` gives the exact
   digits. `formatFmg` uses the same layout with the symbol `Fmg`.

## Sources

- Symbol `Ar`, name `Ariary`, symbol spacing: Unicode CLDR 49, locale `mg`
  ([currencies.json](https://raw.githubusercontent.com/unicode-org/cldr-json/main/cldr-json/cldr-numbers-full/main/mg/currencies.json),
  [numbers.json](https://raw.githubusercontent.com/unicode-org/cldr-json/main/cldr-json/cldr-numbers-full/main/mg/numbers.json)).
- Space as thousand separator: Ministry of Economy and Finance, _Annexe au projet de loi
  n° 022/2024 portant loi de finances pour 2025, Tome 3_ (amounts such as `12 075 821 312`),
  [PDF](https://cabri-sbo.org/uploads/bia/Madagascar_2025_Formulation_External_BudgetFramework_MinistryOfFinanceAndEconomicDevelopment_SADCCOMESA_French_32db46.pdf).
- Code `MGA`, numeric 969: ISO 4217 list one (SIX, published 2026-09-17),
  [XML](https://www.six-group.com/dam/download/financial-information/data-center/iso-currrency/lists/list-one.xml).
- Franc withdrawn after 2004-12-31: CLDR
  [currencyData.json](https://raw.githubusercontent.com/unicode-org/cldr-json/main/cldr-json/cldr-core/supplemental/currencyData.json)
  (MG: `MGF` until 2004-12-31).
- 1 ariary = 5 francs: Malagasy Dictionary, [ariary](https://malagasyword.org/bins/teny2/ariary)
  ("5,00frs"). The Banky Foiben'i Madagasikara site gave no readable page when checked.

## Recorded decisions

- **Space or dot, never a comma.** CLDR `mg` groups thousands with a comma (`Ar 1,500`). The
  project lead decided (2026-11-24) that Madagascar uses a space or a dot, which matches the
  Ministry of Finance documents above. The spec follows the space by default, offers the dot
  as an option, and keeps CLDR's symbol position (`Ar` first).
- **Reading separators.** Text pasted from documents uses spaces, no-break spaces or narrow
  no-break spaces interchangeably, so all three count as the same separator when reading.
- **Separator options, Malagasy by default.** Every option defaults to the Malagasy way; other
  conventions are opt-in, so a caller reading text from elsewhere says so instead of the
  library guessing. Decided by the project lead on 2026-11-26.
- **Decimals, with a comma.** Amounts take any number of decimals, written after a comma
  (`Ar 1 500,5`), as in French and Malagasy writing; the dot stays a thousand separator. Reading
  `1,500` therefore gives 1.5: in Madagascar it is not ambiguous. Decided by the project lead on
  2026-11-26, confirmed 2026-11-27.
- **No silent rounding.** Reading and converting never round: a value a number cannot hold is
  `PRECISION_LOSS`, and the caller chooses the exact text with `{ as: 'text' }`. Decided by the
  project lead on 2026-11-27.
- **Symbol `Fmg`.** ISO uses `MGF` and CLDR `mg` has no symbol for the franc; `Fmg` is the
  abbreviation the project lead asked for.

## Not provided

- **FMG amounts in words.** Not provided. `iraimbilanja` names one franc (1/5 ariary) in
  [English Wikipedia, Malagasy franc](https://en.wikipedia.org/wiki/Malagasy_franc) and
  [Malagasy Wikipedia, Ariary](https://mg.wikipedia.org/wiki/Ariary), and the latter says a
  franc amount is spoken in ariary (`2 500 F` is said `dimanjato`). No source spells a whole
  FMG amount, so the spec adds no FMG words function until one does. Decided by the
  maintainers on 2026-11-26.

## Recorded disagreements

- **Minor unit.** ISO 4217 lists 2 minor units for MGA; CLDR uses 0 digits and amounts are
  usually written without decimals. Since spec 0.5.0 the spec follows neither limit: amounts
  take any number of decimals (see "Recorded decisions").
- **Thousand separator.** See "Recorded decisions": CLDR comma vs Ministry of Finance space.
