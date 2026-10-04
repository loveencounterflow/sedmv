
# `sedmv` – rename files with an action string

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [`sedmv` – rename files with an action string](#sedmv--rename-files-with-an-action-string)
  - [Purpose](#purpose)
  - [Usage](#usage)
  - [Action string (AS)](#action-string-as)
    - [Examples](#examples)
  - [FIND](#find)
  - [REPLACE](#replace)
    - [Entities](#entities)
  - [Which part of the name?](#which-part-of-the-name)
  - [Length limit](#length-limit)
  - [Flow and safety [proposal]](#flow-and-safety-proposal)
  - [Non-goals](#non-goals)
  - [Future](#future)
    - [Command variants (find syntaxes)](#command-variants-find-syntaxes)
    - [Other ideas](#other-ideas)
  - [Open points](#open-points)
  - [Existing tools (research)](#existing-tools-research)
  - [Implementation and references](#implementation-and-references)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# `sedmv` – rename files with an action string

> **Status:** design draft from the design discussion.
> **[decided]** = settled by the author. **[proposal]** = still to be confirmed.

## Purpose

Bulk renaming on Linux as a replacement for Bulky (Nemo): search/replace with regular expressions, plus
placeholders ("entities") for counters and content hashes. Main use case: give filenames a short, readable
content-hash signature, e.g. `%387a6c%blah.txt` → `€xx-xxx-xxx;;blah.txt`.

## Usage

```bash
sedmv [options] ACTION-STRING FILE...
```

Always put the action string in single quotes in the shell (`&`, `;`, `$`, `*` and `^` are shell syntax otherwise).

## Action string (AS)

Structure [decided]:

```
CMD DELIM FIND DELIM REPLACE DELIM FLAGS
```

* **CMD** matches `/^[0-9a-zA-Z]*/` and is optional. The default is `s` (substitute), for now the only
  command. `'s/a/b/'` and `'/a/b/'` are equivalent.

* **DELIM** is the first character after CMD. Any character except ASCII letters and digits is allowed. For
  `s` it occurs exactly three times.

* **The delimiter cannot be unescaped.** When splitting, a backslash plus the following character is skipped
  and passed on unchanged; what that means is up to the regex or the replacement text. Choosing `(` as
  delimiter means giving up `()` as groups; that is the user's responsibility. Example: `'s(\(([(g'` is the
  same as `'s/\(/[/g'` (replace every `(` in the name with `[`).

* **FLAGS** are regex flags for the FIND part, e.g. `i` [decided].


[proposal] More flags: `g` (all matches; the default is the first match only), `s`, `m`, `x` (extended
syntax, see below). `n` and `v` are always on, `u` is an error.

[proposal] Reject `\` as a delimiter. If the number of delimiters is wrong, the error message names the
delimiter and the count found.

[proposal] Reserve `-` for command variants (see [Future](#future)): CMD would then match
`/^[0-9a-zA-Z-]*/`, and `-` can no longer be a delimiter. Doing this from the start avoids a breaking change
later, because `'s-a-b-'` would otherwise change meaning.

### Examples

```bash
# Prefix a three-digit counter:  photo.jpg -> 001_photo.jpg
sedmv 's/^/&n%000;_/' *

# Replace an existing %xxxxxx% hash or insert a new one:
#   %387a6c%blah.txt -> €xx-xxx-xxx;;blah.txt
sedmv 's/^(%[0-9a-f]{6}%)?/€&ch%00-000-000;;;/i' *

# Extension only: .jpeg -> .jpg
sedmv -x 's/^jpeg$/jpg/' *

# Whole name, e.g. for multi-part extensions
sedmv -w 's/\.tar\.gz$/.tgz/' *
```

## FIND

JavaScript RegExp, prepared with [`regex`](https://github.com/slevithan/regex) (Steven Levithan):

* **Always on:** `v` (UnicodeSets) and `n` (named capture only; unnamed `(…)` do not capture). There are no
  numbered groups or backreferences.

* Also available through `regex`: atomic groups, possessive quantifiers, subroutines.

* The emulated flag `x` (ignore whitespace and `#` comments in the pattern) is **off** by default, because
  filenames contain spaces. It can be switched on with the AS flag `x` [proposal].

* `v` is stricter than `u`: inside character classes `-`, `(`, `)`, `[`, `]`, `{`, `}`, `/`, `|` and `\`
  must be escaped, and doubled punctuation (including `;;`) is reserved. `[\w-]` is an error, `[\w\-]` is
  not. The tool should print the `SyntaxError` together with a hint [proposal].

* Requires Node.js ≥ 20 (native `v` support).


## REPLACE

The replacement text consists of literals, **entities** and JavaScript replacement patterns.

**JavaScript patterns [decided]:** whatever JavaScript accepts at this place (`$<name>` for named groups,
`$&`, `$$`, …). Because of `n` there is no `$1`; it stays literal text. Named groups are written `$<name>`,
so they cannot collide with entities.

Two consequences for the implementation [proposal]:

* Entities are expanded before the substitution. Their results are `$`-escaped (`$` → `$$`) so JavaScript
  does not interpret them a second time.

* `$<name>` is checked against the groups of FIND at startup. Otherwise JavaScript silently replaces unknown
  names with nothing.

### Entities

Syntax: `&NAME;` or `&NAME%ARGUMENT;`. The argument extends to the first `;`. A `;` right after that is an
ordinary character: `&ch%00-000-000;;;` is the entity followed by the literal `;;` [decided].

The set of entities is small and closed. HTML entities are not supported (if ever, with a namespace prefix
such as `&h:auml;`).

| Entity | Meaning |
|---|---|
| `&n%000;` | Counter. Each `0` stands for one decimal digit (with leading zeros). Counting follows the order of the file arguments. [decided] |
| `&ch%00-000-000;` | Content hash of the file. Each `0` stands for one digit (here 8 digits = 40 bits); all other characters in the argument are literals. [decided] |

**Hash alphabet [decided]:** Crockford base32, lowercase: `0123456789abcdefghjkmnpqrstvwxyz` (no `i l o u`).

[proposal] Hash function: SHA-256 over the file contents, most significant bits.

[proposal] Further rules:

* Entity names are lowercase. A `&` that does not form a valid entity (e.g. a missing `;`) is an error. A
  literal `&` is written `&amp;`.

* Numeric character references `&#48;` / `&#x20AC;` for arbitrary characters.

* `&date…;` (mtime) takes a picture string such as `YYYY-MM-DD` as its argument, not strftime, because `%`
  already introduces the argument. Open: mtime or ctime, time zone.

## Which part of the name?

[decided]

| Option | Effect |
|---|---|
| (none) | FIND/REPLACE act on the **stem**. The extension is left unchanged and re-appended. |
| `-x`, `--extension` | acts on the **extension** only (without the dot) |
| `-w`, `--whole-name` | acts on the **whole name**; the extension is not treated specially. Excludes `-x`. |

Note: the command-line option `-x` has nothing to do with the AS flag `x`.

**What is an extension?** [proposal] The name is split at the last ASCII dot. There is an extension only if

1. the dot is neither the first nor the last character,

2. the stem does not consist of dots only,

3. the text after the dot contains no whitespace.

Otherwise the name consists of the stem only (`.bashrc`, `foo.`, `Dr. Müller`). When reassembling, the dot
is added only if the extension is non-empty. Files without an extension are skipped under `-x`. Known
limits: `archive.tar.gz` has the extension `gz`, and `Lettre à M.Dacier` is taken to have the extension
`Dacier`. Use `-w` for such cases.

## Length limit

[carried over from the Bash prototype, not yet discussed for the AS model] The limit is 255 **bytes** (not
characters). If a name gets too long, the stem is shortened (at a UTF-8 character boundary) and the
extension is kept. Option `-t ask|yes|no`.

## Flow and safety [proposal]

* First a preview (old → new) and a single confirmation for the whole batch. `-y` skips it, `-n` is a dry
  run.

* Collisions (within the batch and with existing files) are detected before the first rename. Chains and
  cycles (`a→b`, `b→a`) go through temporary names.

* Only regular files and only basenames are processed. A `/` in the result is an error.


## Non-goals

Checking hashes against changed file contents is a different topic from renaming. Git is responsible for
change management, and large binary files can get a dedicated tool (maybe a plugin later). The renamer is
therefore only as idempotent as the FIND pattern makes it (see example 2).

## Future

Ideas only; nothing in this section is decided.

### Command variants (find syntaxes)

The `s` command (and other commands) should be extensible. A hyphenated suffix selects the syntax of the
FIND part:

| Command | FIND syntax |
|---|---|
| `s`, `sub` | regular expression (default) |
| `s-lit`, `sub-lit` | literal text: no metacharacters, nothing to escape |
| `s-wild`, `sub-wild` (short: `s-w`) | shell-style wildcards (`*`, `?`), compiled to a regex |

Consequences:

* `-` becomes part of the command alphabet and can no longer be a delimiter (see the proposal under *Action
  string*).

* Backslash handling differs: in `-lit` there is no regex to receive `\X`. [proposal] Backslash has no
  special meaning there; choose a delimiter that does not occur in FIND.

* Flags: `i` and `g` apply; regex-only flags (`s`, `m`, `x`) are errors in these variants. In REPLACE,
  entities and `$&` keep working; `$<name>` is an error because there are no named groups. How wildcard
  matches can be referenced in REPLACE is open.

* [proposal] Prefer the long forms `s-lit` / `s-wild` over the one-letter `s-w`: `-w` is already
  `--whole-name` on the command line, which invites confusion.

### Other ideas


* `y` command: transliteration as in sed (e.g. spaces → underscores)

* Several actions applied in sequence (e.g. repeated `-e AS`)

* More entities: `&date…;` (mtime/ctime), parent directory name, file size

* Hash verification / change detection as a plugin (a non-goal today)

* Undo log: record every rename so a batch can be reverted

* HTML entities with namespace prefix (`&h:auml;`)

* Recursion into directories

## Open points

* flag list for the AS; ban `\` as a delimiter? reserve `-` now?

* hash function; start value and step of the counter

* `&date…;`: time source, time zone, picture string; further entities

* validation of `$<name>`; handling of `x` and `u`

* length limit: option and behaviour in the AS model

* `-w` as an option name


## Existing tools (research)

None of these offers base32/base36 signatures, regex action strings or entities.

* hash_rename (Python), hex hash as prefix: https://github.com/DeflateAwning/hash_rename

* affix-hash (npm), selectable algorithm and length, hex: https://npmjs.org/package/affix-hash

* nym (Go), timestamp + BLAKE3: https://pkg.go.dev/github.com/kirinyoku/nym@v1.2.1

* renfnv (Go), FNV-1a in base32 instead of the original name: https://pkg.go.dev/git.sr.ht/~kisom/goutils/cmd/renfnv

* Bulky (Nemo), GUI, model for find/replace: https://www.github.com/linuxmint/bulky


## Implementation and references

Node.js ≥ 20. Regex preparation with `regex`: https://github.com/slevithan/regex. `regex-utilities`
(https://github.com/slevithan/regex-utilities) is only needed if FIND itself is preprocessed, e.g. entities
inside the pattern.

Predecessor: a Bash prototype `hashname` (format `&xxxx-xxxx:name`, with signature verification). It is not
part of this design.
