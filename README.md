# too-many-th

```
 ████████╗ ██████╗  ██████╗     ███╗   ███╗ █████╗ ███╗   ██╗██╗   ██╗    ████████╗██╗  ██╗
 ╚══██╔══╝██╔═══██╗██╔═══██╗    ████╗ ████║██╔══██╗████╗  ██║╚██╗ ██╔╝    ╚══██╔══╝██║  ██║
    ██║   ██║   ██║██║   ██║    ██╔████╔██║███████║██╔██╗ ██║ ╚████╔╝        ██║   ███████║
    ██║   ██║   ██║██║   ██║    ██║╚██╔╝██║██╔══██║██║╚██╗██║  ╚██╔╝         ██║   ██╔══██║
    ██║   ╚██████╔╝╚██████╔╝    ██║ ╚═╝ ██║██║  ██║██║ ╚████║   ██║          ██║   ██║  ██║
    ╚═╝    ╚═════╝  ╚═════╝     ╚═╝     ╚═╝╚═╝  ╚═╝╚═╝  ╚═══╝   ╚═╝          ╚═╝   ╚═╝  ╚═╝
```

---

## ◆ PULSE

[![GitHub Pages](https://img.shields.io/badge/Pages-live-2ea44f)](https://suradet-ps.github.io/too-many-th/)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](#-anatomy)

A linked list has one head, and one nightmare - too-many-th is the Thai
bridge to that exact nightmare. This is the complete Thai translation of
the official `Learning Rust With Entirely Too Many Linked Lists` book: 58
pages built with mdbook, terminology locked by a single glossary, and
every code block byte-identical to the original. The links are checked
against the built book, the structure mirrors the upstream repo
file-for-file, and the license travels with the text. Built for the
Thai-speaking student of Rust:
[suradet-ps.github.io/too-many-th](https://suradet-ps.github.io/too-many-th/).

| แปลครบ 58 หน้า ▣ | Glossary ▣ | ลิงก์ทั้งหมดผ่าน ▣ | Build ผ่าน ▣ |
|---|---|---|---|

*v1.0.0 - translation, glossary, verification, and the static build
are all sealed.*

> Built with mdbook 0.5 + Markdown, translated from
> [rust-unofficial/too-many-lists](https://github.com/rust-unofficial/too-many-lists),
> verified by script and rendered as static HTML - six linked lists, six
> chapters, and one bonus chapter of nonsense.
>
> **suradet-ps**, artifact keeper

---

## ◆ IGNITION

One runtime, three commands.

```
⟫ git clone https://github.com/suradet-ps/too-many-th.git
⟫ cd too-many-th
⟫ cargo install mdbook
⟫ mdbook serve --open
```

Open [http://localhost:3000](http://localhost:3000).

```
⟫ mdbook build                                    # static HTML into book/
⟫ powershell scripts/check-links.ps1              # all anchors in the built book (pwsh on Linux/macOS)
⟫ powershell scripts/verify-translation.ps1       # byte-exact check vs upstream
```

> On Linux or macOS, run the verification scripts using `pwsh scripts/<script>.ps1`.
> `verify-translation.ps1` checks against `too-many-lists` in adjacent directories or via `-Orig <path>`.

<details>
<summary>Translating a chapter</summary>

A chapter is a file: `src/<chapter>.md`, listed in `src/SUMMARY.md`.
The glossary lives in `GLOSSARY.md` - a term is chosen once and reused
everywhere. Code blocks, commands, links, and filenames stay verbatim;
only prose and headings are translated. Heading anchors follow mdbook's
slug rules (Thai tone marks are stripped), so anchors are copied from
the built HTML, never guessed.

</details>

---

## ◆ ANATOMY

One stack of lists, zero custom JS, several quiet helpers.

- **Translates** - the complete book: six chapters of linked lists - a
  bad stack, an ok stack, a persistent stack, a bad safe deque, an ok
  unsafe queue, a production unsafe deque - and a bonus chapter of silly
  lists - Thai prose over untouched code.
- **Glossaries** - `GLOSSARY.md` locks the vocabulary (ownership =
  ความเป็นเจ้าของ, node = โหนด, reference = เรเฟอเรนซ์), so the sixth
  chapter agrees with the first.
- **Verifies** - `scripts/verify-translation.ps1` diffs every code
  block, heading level, and link target against upstream
  `too-many-lists` - byte-exact or it does not pass.
- **Checks** - `scripts/check-links.ps1` walks the built book and
  resolves every anchor link against real heading ids - all reachable.
- **Builds** - mdbook renders static HTML into `book/`, zero server
  runtime, readable offline and searchable by built-in static index.
- **Licenses** - MIT, inherited from upstream, with the LICENSE file
  shipped beside the text.

---

## ◆ RITUALS

**The core ceremony** - the translation pass:

1. Open a chapter in `src/`. The upstream `too-many-lists` repo sits
   beside it (clone `https://github.com/rust-unofficial/too-many-lists`
   alongside `too-many-th`) - structure is a contract.
2. Translate the prose; keep every code block and command as the
   original wrote it.
3. Consult `GLOSSARY.md` for every term that already has a canon. New
   terms get proposed in the glossary first.
4. Build, verify, check. The book builds clean, the diff is byte-exact,
   and the anchors resolve.

**The ceremony of the anchor** - mdbook slugs strip Thai tone marks
(`การจับคู่แพตเทิร์น` becomes `การจับคูแพตเทิรน`). Anchors are read from
the built HTML, written into the source, and re-verified - a guessed
anchor is a broken link waiting to happen.

**The ceremony of the code block** - a translated command that is not
byte-identical to the original is a regression, not a translation. The
verifier is the conscience of the repo.

---

## ◆ ECHOES

**Where this artifact is heading**

```
P1 ▸ SUMMARY + introduction, A Bad Stack, An Ok Stack ────────────── ▸ sealed
P2 ▸ A Persistent Stack, A Bad Safe Deque ────────────────────────── ▸ sealed
P3 ▸ An Ok Unsafe Queue, A Production Unsafe Deque ────────────────── ▸ sealed
P4 ▸ A Bunch of Silly Lists, glossary, license, link verification ─── ▸ sealed
```

**Raising the artifact** - the honest path lives in `GLOSSARY.md`
(term canon), `scripts/` (the verification gate), and `book.toml`
(book config). New chapters follow the frontmatter-free contract of the
SUMMARY. Open an issue first to discuss a change.

**Status** - on every change: `mdbook build` must pass, the translation
verifier must report byte-exact code blocks across all 59 files (58
pages + `SUMMARY.md`), and the link checker must report
`ALL ANCHOR LINKS OK`. [Watch the gates](scripts).

---

```
  ─────────────────────────────────────────
   ทุกลิงก์ลิสต์มีหัวของมัน
   ทุกหนังสือมีสแต็กของมัน
  ─────────────────────────────────────────
```

Translated from the [too-many-lists](https://github.com/rust-unofficial/too-many-lists)
book, which is licensed under [MIT](license-MIT). Copyright (c) 2015
Aria Desires.