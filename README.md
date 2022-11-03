# rsvp-synsets

A way to quickly read open parts of speech organized by synset groupings.

<strong>Built with:</strong>
[SvelteKit](https://github.com/sveltejs/kit), [english-wordnet](https://github.com/globalwordnet/english-wordnet) and [fast-xml-parser](https://github.com/NaturalIntelligence/fast-xml-parser).

<strong>Styled with:</strong>
[radix-icons-svelte](https://github.com/Brisklemonade/radix-icons-svelte)

## Goal

Rapid Serial Visual Presentation is a technique for timing the display of images. A program serves content very quickly, which forces a person to break their own subvocalization habits. One use case is for speed reading. When words are shown on a screen, each display for short intervals. The newest word replaces the previous word where the latter was placed. In this way, a reader's eye does not move as much as it would if they were scrolling lines of text.

Synsets are a set of one or more synonyms. A wordnet combines synsets and orders them logically. If two words share some relation, it ought to be faster to learn about them as a group. So to speed-up the process of going from synset to synset, I apply RSVP to move through a comprehensive graph of synsets made by [globalwordnet](https://github.com/globalwordnet).

The demo.