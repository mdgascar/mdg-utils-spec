# date: Malagasy dates

Writes Gregorian dates in Malagasy, reads them back, says the time of day in words, and gives the
Malagasy names of months and weekdays.

## Public API

Every language port provides each function below, named the way its language does it. Behaviour
is set by the rules and test vectors here. Parsers (text typed by users → value) come in three forms:
`parse` throws, `safeParse` returns the value or the error code, `tryParse` returns the value or
nothing.

| Function        | Does                                            | JavaScript                       | PHP                                               |
| --------------- | ----------------------------------------------- | -------------------------------- | ------------------------------------------------- |
| `formatDate`    | Date → text in a `DateStyle`                    | `formatDate(date, style)`        | `MalagasyDate::fromString($date)->format($style)` |
| `parseDate`     | Malagasy date text → date, or throws            | `parseDate(text, style)`         | `MalagasyDate::parse($text, $style)`              |
| `safeParseDate` | Text → date or error code, never throws         | `safeParseDate(text, style)`     | `MalagasyDate::safeParse($text, $style)`          |
| `tryParseDate`  | Text → date, or nothing, never throws           | `tryParseDate(text, style)`      | `MalagasyDate::tryParse($text, $style)` → `null`  |
| `spellTime`     | Time → Malagasy words                           | `spellTime(time)`                | `MalagasyTime::fromString($time)->spell()`        |
| `parseTime`     | Words → time, or throws                         | `parseTime(text)`                | `MalagasyTime::parse($text)`                      |
| `safeParseTime` | Words → time or error code, never throws        | `safeParseTime(text)`            | `MalagasyTime::safeParse($text)`                  |
| `tryParseTime`  | Words → time, or nothing, never throws          | `tryParseTime(text)`             | `MalagasyTime::tryParse($text)` → `null`          |
| `monthName`     | `1`–`12` → month name, wide or abbreviated      | `monthName(month, width)`        | `MalagasyDate::monthName($month, $width)`         |
| `weekdayName`   | ISO weekday `1`–`7` → name, wide or abbreviated | `weekdayName(isoWeekday, width)` | `MalagasyDate::weekdayName($weekday, $width)`     |

Allowed per language, because they adapt native types: building a date from the language's own
date type (`Date` in JavaScript, `DateTimeInterface` via `MalagasyDate::fromDateTime()` in PHP)
and reading the ISO weekday of a date object (`MalagasyDate->isoWeekday()` in PHP).

## Rules

1. **Input** is a calendar date, never a time:
   - text in ISO 8601 form `YYYY-MM-DD` (`2026-11-23`), or
   - calendar fields `{ year, month, day }`; ports may also accept their native date type
     (JavaScript `Date`, read in its local calendar fields; PHP `DateTimeInterface`).
2. **Years 1 to 9999.** Other years are `OUT_OF_RANGE`.
3. **Impossible dates** (`2026-02-30`, `2023-02-29`): `INVALID_FORMAT` for text,
   `OUT_OF_RANGE` for calendar fields. Text in any other shape (`26/06/1960`, `2026-1-5`, with a
   time) is `INVALID_FORMAT`. Anything else is `INVALID_TYPE`.
4. **Styles** (`formatDate`), default `long`:

   | Style     | Pattern         | `2026-11-23`                   |
   | --------- | --------------- | ------------------------------ |
   | `long`    | `d MMMM y`      | `23 Novambra 2026`             |
   | `full`    | `EEEE d MMMM y` | `Alatsinainy 23 Novambra 2026` |
   | `medium`  | `y MMM d`       | `2026 Nov 23`                  |
   | `short`   | `y-MM-dd`       | `2026-11-23`                   |
   | `numeric` | `dd/MM/y`       | `23/11/2026`                   |

   Pattern letters: `EEEE` weekday name, `d` day, `dd` two-digit day, `MMMM` month name, `MMM`
   abbreviated month, `MM` two-digit month, `y` year. Any other style is `INVALID_TYPE`.

5. **Reading dates** (`parseDate`): any style of rule 4, so every date `formatDate` writes
   reads back: `23 Novambra 2026`, `23/11/2026` → `{ year: 2026, month: 11, day: 23 }`. Case and
   spacing do not matter, and day and month may have one digit (`5/7/2026`). A `full` date
   must name its own weekday. A day that does not exist is `INVALID_FORMAT`, a year outside
   1–9999 `OUT_OF_RANGE`, text over 64 characters `INVALID_FORMAT`. With a `style`, only that
   style is read. Without one, a text two styles read as different dates is `AMBIGUOUS`, never
   guessed: `5 Mey 12` is 5 May of year 12 (`long`) or 12 May of year 5 (`medium`), since
   `Mey` is both the name and the abbreviation. This only happens for years up to 31.

6. **Names.** Months 1–12, weekdays numbered as in ISO 8601 (Monday 1 … Sunday 7). Width
   `wide` (default) or `abbreviated`; other widths are `INVALID_TYPE`, other numbers
   `OUT_OF_RANGE`.

| #   | Month     | Abbr. |     | ISO | Weekday     | Abbr. |
| --- | --------- | ----- | --- | --- | ----------- | ----- |
| 1   | Janoary   | Jan   |     | 1   | Alatsinainy | Alats |
| 2   | Febroary  | Feb   |     | 2   | Talata      | Tal   |
| 3   | Martsa    | Mar   |     | 3   | Alarobia    | Alar  |
| 4   | Aprily    | Apr   |     | 4   | Alakamisy   | Alak  |
| 5   | Mey       | Mey   |     | 5   | Zoma        | Zom   |
| 6   | Jona      | Jon   |     | 6   | Asabotsy    | Asab  |
| 7   | Jolay     | Jol   |     | 7   | Alahady     | Alah  |
| 8   | Aogositra | Aog   |     |     |             |       |
| 9   | Septambra | Sep   |     |     |             |       |
| 10  | Oktobra   | Okt   |     |     |             |       |
| 11  | Novambra  | Nov   |     |     |             |       |
| 12  | Desambra  | Des   |     |     |             |       |

## Telling the time

**Rule 7.** **Time** (`spellTime`): `amin'ny`, the hour on a 12-hour clock in words, the minutes, then the
part of the day: `14:30` → `amin'ny roa sy sasany tolakandro`. Input is `HH:MM` text (00:00
to 23:59, the hour may have one digit), `{ hour, minute }`, or a port's native time. Other
text is `INVALID_FORMAT`, an hour outside 0–23 or a minute outside 0–59 `OUT_OF_RANGE`.

| Minutes | Words                             | `2:mm`                                                                                                                     |
| ------- | --------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| 0       | (none)                            | `amin'ny roa`                                                                                                              |
| 1–19    | `sy` and the minutes              | `2:05` → `amin'ny roa sy dimy`                                                                                             |
| 15      | `sy fahefany` (a quarter)         | `amin'ny roa sy fahefany`                                                                                                  |
| 20      | `sy fahatelony` (a third)         | `amin'ny roa sy fahatelony`                                                                                                |
| 21–29   | `sy sasany latsaka` 30 − minutes  | `2:25` → `amin'ny roa sy sasany latsaka dimy`                                                                              |
| 30      | `sy sasany` (a half)              | `amin'ny roa sy sasany`                                                                                                    |
| 31–39   | `sy sasany mahery` minutes − 30   | `2:35` → `amin'ny roa sy sasany mahery dimy`                                                                               |
| 40–59   | next hour, `latsaka` 60 − minutes | `2:40` → `amin'ny telo latsaka fahatelony`, `2:45` → `amin'ny telo latsaka fahefany`, `2:55` → `amin'ny telo latsaka dimy` |

The part of the day is that of the hour the time names, as in French: `maraina` 1–11,
`atoandro` 12, `tolakandro` 13–17, `hariva` 18–23, `alina` 0 (said `roa ambin'ny folo`).
`11:45` → `amin'ny roa ambin'ny folo latsaka fahefany atoandro`.

**Rule 8.** **Reading the time** (`parseTime`): the words `spellTime` says, in any case and spacing,
with or without `amin'ny`, back to `{ hour, minute }`. Every time of day has its own words.
Anything else (no part of the day, other words, text over 128 characters) is
`INVALID_FORMAT`.

## Sources

- Clock time: _Malagasy-English/English-Malagasy Dictionary and Phrasebook_ (J. Rasoloson,
  Hippocrene), as quoted by [Wild Madagascar](https://www.wildmadagascar.org/people/malagasy-time.html):
  `Amin'ny roa` (2:00), `sy dimy` (2:05), `sy sasany` (2:30), `Amin'ny telo latsaka fahefany`
  (2:45), `latsaka dimy` (2:55). Dictionary entries [maraina](https://malagasyword.org/bins/teny2/maraina)
  (`Amin' ny folo maraina`) and [hariva](https://malagasyword.org/bins/teny2/hariva)
  (`Amin' ny dimy hariva`). Parts of the day: Peace Corps Madagascar, _Malagasy Language
  Lessons_.

- Month and weekday names, abbreviations, and the `long`, `full`, `medium` and `short`
  patterns: Unicode CLDR 49, locale `mg`,
  [ca-gregorian.json](https://raw.githubusercontent.com/unicode-org/cldr-json/main/cldr-json/cldr-dates-full/main/mg/ca-gregorian.json).
- Cross-checked with the Malagasy Dictionary:
  [Janoary](https://malagasyword.org/bins/teny2/Janoary),
  [Alatsinainy](https://malagasyword.org/bins/teny2/Alatsinainy) ("andro faharoa amin'ny
  herinandro", second day of the week), [Alahady](https://malagasyword.org/bins/teny2/Alahady)
  ("andro voalohany amin'ny herinandro", first day of the week).

## Recorded decisions

- **Minutes and parts of the day (rule 7).** The sources give the hour, the quarter, the half and
  `latsaka` before the next hour. The project lead set the rest on 2026-11-27: `fahatelony` for 20
  and 40, `sasany latsaka` and `sasany mahery` around the half hour, `latsaka` from 40, and the
  parts of the day by the French hours (matin, midi, après-midi, soir, minuit).

- **`numeric` style `dd/MM/y`.** CLDR `mg` has no such pattern (its `short` is `y-MM-dd`). The
  project lead asked (2026-11-24) for both: CLDR by default, and `dd/MM/y` as an option because
  it is the everyday written form in Madagascar.
- **ISO weekday numbers.** The dictionary counts Sunday (Alahady) as the first day of the week;
  the spec numbers weekdays as ISO 8601 (Monday 1) so every language maps them the same way.
  Names are unaffected.
