---
name: password-generator
description: Generates secure, memorable passwords using 3 random dictionary words with leet-speak substitutions. Use when the user asks to generate a password, create a new password, or needs a secure passphrase.
---

# Password Generator

Generate 5 memorable yet secure passwords using 3 random dictionary words with character substitutions.

## Rules

1. Pick 3 random common English words (nouns, verbs, adjectives — vary the parts of speech)
2. Apply random leet-speak substitutions to some (not all) characters:
   - `a` → `4` or `@`
   - `s` → `$`
   - `h` → `#`
   - `l` → `!`
   - `e` → `3`
   - `o` → `0`
   - `i` → `1`
3. Randomly capitalize one word (or the first letter of each word)
4. Join words without spaces (or with a separator like `-` or `.`)
5. Each password must be **at least 12 characters** and contain:
   - Lowercase letters
   - Uppercase letters
   - At least one special character (`@`, `$`, `#`, `!`, `4`, `0`, `3`, `1`, `.`, `-`, etc.)

## Output Format

Present exactly 5 passwords as a numbered list with no extra commentary:

```
1. Gr@pe$toneBrick
2. fl0werC!oudmatch
3. T0ast#ammer-wind
4. kn1fe$hadowBlaze
5. Purp!3-rock-sn4ke
```

## Generation Strategy

- Vary word length: mix short (3-5 chars) and longer (6-9 chars) words so total hits 12+
- Don't substitute every eligible character — keep passwords readable/memorable
- Vary separators: some passwords use no separator, some use `-` or `.`
- Apply capitalization differently across the 5 passwords (first word, last word, random word, title case, all-lower-except-subs)
- Never repeat the same word across all 5 passwords
- Avoid offensive or sensitive words
