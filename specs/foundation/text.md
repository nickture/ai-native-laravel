---
paths:
  - "resources/js/**"
  - "resources/views/**"
  - "specs/**/*.md"
---

# Overy — Text

> Decisions about Overy’s copy: readers, languages, tone, vocabulary and formatting. Russian text follows the rules of the `nickture-text-ru` skill unless this file says otherwise. Design — `specs/foundation/design.md`. Which layer takes precedence — `specs/foundation/project.md`, section “How project knowledge is organized”.

## Readers and languages

| Text | Reader | Language |
|---|---|---|
| Interface, changelog, landing, `/legal`, Telegram bot replies | User | English |
| `specs/`, plans, `.claude/CLAUDE.md` | Owner and agents | Russian, code terms as in the code |
| Code, comments, commits, `.ai/rules` | Developer and agents | English |

There is no translation layer, and none is planned: the app locale is `en`. User content can be in any language, and the layout holds long and Cyrillic titles.

## Interface tone

- Present tense. Subject, verb, object. No metaphors.
- Name the result the user sees, not the mechanism.
- Sentence case: buttons, headings, menu items (`Add a property`, `Clear selection`).
- A quoted interface string in a document or the changelog matches the string in the code.
- The bot’s reply to a capture is short: `✅ Added`, without retelling what was saved.
- Marketing: “second brain” is lowercase in running text. A label above a section heading appears only if it says something the heading itself does not.

## Document tone

Businesslike. A document describes the current state: no history, release dates, “it used to be” or options to roll back to old versions. Before each sentence, ask: does it help write the code correctly?

## Vocabulary

| Interface word | Meaning | Do not use |
|---|---|---|
| node | A unit of the vault, any record | — |
| note | A plain node without a state: the Notes view, a capture that landed as a note | For any node in general, or for a file on disk |
| actionable | A node with the actionable trait; it has a state | task, to-do, checklist item |
| state | The state of an actionable node: Inbox, Next, Waiting, Someday, Done, Cancelled and the user’s own | status |
| type | A label marked as a type (Book, Person, Project). Actions — “Make this a type”, “Make not a type” | category, kind |
| label | Any label on a node: tag, context, type, `#` and `@` in text. Actions — “Make this a label”, “Make not a label” | classify, classifier, mention |
| Top | The tree root, the mountain glyph. Strings — `ROOT_LABEL` and `ROOT_SUBLABEL` in `moveParentPatch.ts` | Root, Vault |
| Second Brain | The user’s whole space in the interface: page title, settings | Vault |
| Labels / Labeled by | A node’s outgoing and incoming labels (MetaPanel sections, Summary cards) | Mentions out, Mentioned in |
| External labels | A Summary card: what the subnodes are labeled with | External mentions |
| Locate | Jump to the token in the editor | Find in note |

- Code names are not renamed to match the interface: `Vault*`, `mention`, `classifier` stay in the code and in documents, but do not reach the interface.
- A new word for the user appears only with the owner’s consent. An existing word is looked for first.

## Formatting

- Headings in a node body are real `###`, not a bold line.
- Dates in the interface: the past as relative time, the future as a short date in the browser locale, an empty value as `—` (`resources/js/lib/dateDisplay.ts`).
- Placeholder names in examples, demos and the changelog are neutral: @Alex, not @Vasya. Cyrillic node titles are not quoted in the changelog.

## Changelog

The changelog rules are in the header of `resources/js/lib/changelogReleases.ts`: whoever edits that file reads them. The vocabulary there links here.
