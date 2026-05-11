# Stretchy Words — Regular Expressions Assignment

---

## 1. Requirements

The problem asks us to determine how many words in a list are **stretchy** relative to a source string `S`.

A word is stretchy if it can be "extended" to become equal to `S` by repeating characters in a group until that group has **3 or more** characters. 

The key rules are:
- Both S and the word must have the **same character groups in the same order** (e.g., `h`, `e`, `l`, `o`)
- For each group, either the counts match exactly, **or** S's group count is ≥ 3 and larger than the word's group count
- A group of size 1 or 2 in S **cannot** have been stretched — it must match exactly

---

## 2. How I Solved It

The solution uses **Python with `pandas` and `re` (regular expressions)** to process all words at once using vectorized operations.

I broke the problem into three conditions that a word must satisfy to be stretchy:
1. It must have the **same number of character groups** as S
2. It must have the **same characters in the same order** as S
3. For each group, the counts must either **match exactly**, or S's group must be **size ≥ 3** and larger (meaning the word's smaller group could have been stretched up to S)

### Step 1 — Read input as a Series
```python
word_list = pd.Series(input().split())
```
The first word is treated as `S`; all others are query words.

### Step 2 — Extract character groups with regex
```python
char_list = word_list.apply(lambda x: re.findall(r'(.)\1*', x))
```
The regex `(.)\1*` captures each run of identical characters:
- `(.)` — captures any single character
- `\1*` — matches zero or more repetitions of that same character (backreference)

For `"heeellooo"` this gives `['h', 'e', 'l', 'o']`.

### Step 3 — Count each group's length
```python
char_count = word_list.apply(
    lambda x: [len(m.group()) for m in re.finditer(r'(.)\1*', x)]
)
```
`re.finditer` returns match objects so we can measure each group's length.  
For `"heeellooo"` this gives `[1, 3, 2, 3]`.

### Step 4 — Apply the stretchy condition
```python
is_stretchy = word_list.index.to_series().apply(
    lambda i:
        len(char_list[0]) == len(char_list[i]) and
        char_list[0] == char_list[i] and
        all(
            c0 == ci or (c0 >= 3 and c0 > ci)
            for c0, ci in zip(char_count[0], char_count[i])
        )
)
```
To keep the logic clean, I broke the stretchy check into **three separate conditions** that must all be true. Failing any one of them immediately returns `False`:

**Condition 1 — Same number of groups:**
```python
len(char_list[0]) == len(char_list[i])
```
Ensures both words have the same number of character groups. This guards against `zip` silently stopping early on a shorter word (e.g. `"hee"` vs `"heeellooo"` — without this check, zip would only compare the first 2 groups and miss the missing `l` and `o`).

**Condition 2 — Same characters in the same order:**
```python
char_list[0] == char_list[i]
```
Ensures both words are built from the same letters in the same sequence. For example `"hello"` → `['h','e','l','o']` must match S's groups exactly. This also implicitly re-checks length, but keeping Condition 1 separate makes the intent explicit.

**Condition 3 — Each group count is valid:**
```python
all(c0 == ci or (c0 >= 3 and c0 > ci)
    for c0, ci in zip(char_count[0], char_count[i]))
```
For every pair of group counts `(c0 from S, ci from word)`, one of two things must hold:
- `c0 == ci` — the group is the same size, no stretching needed
- `c0 >= 3 and c0 > ci` — S's group is large enough (≥ 3) and the word's group is smaller, meaning the word's group could have been stretched up to S

### Step 5 — Count stretchy words
```python
print(is_stretchy[1:].sum())   # excludes S itself
```

## 3. Output

Given the input:
```
heeellooo hello hi helo
```

The program produces this DataFrame:

| words     | unique_chars       | char_counts  | is_stretchy |
|-----------|--------------------|--------------|-------------|
| heeellooo | [h, e, l, o]       | [1, 3, 2, 3] | True        |
| hello     | [h, e, l, o]       | [1, 1, 2, 1] | True        |
| hi        | [h, i]             | [1, 1]       | False       |
| helo      | [h, e, l, o]       | [1, 1, 1, 1] | False       |

**Stretchy count: 1**

Explanation:
- `hello` → stretchy: `e(1)→eee(3)` ✓, `ll` matches exactly ✓, `o(1)→ooo(3)` ✓
- `hi` → not stretchy: different character groups entirely
- `helo` → not stretchy: `ll` in S is 2, cannot have been stretched from `l(1)` (would need size ≥ 3)

---

## 4. Finite-State Automaton (FSA) Graph

The FSA below describes the regex `(.)\1*` which matches one run of identical characters (one group). This FSA is applied repeatedly to consume the whole string group by group.

```
         any char (c)          same char (c)
            ┌───┐                  ┌───┐
            │   │                  │   │
  ──► (q0) ──────────► (q1) ◄──────────┘
  START              ACCEPT
                   (match group)
```

States:
- **q0** — Start state. Waiting for the first character of a group.
- **q1** — Accept state. We have matched at least one character `c`. We stay in q1 as long as the next character is the same `c`. When a different character appears (or the string ends), the group is complete and accepted.

Transitions:
| From | Input        | To | Action               |
|------|--------------|----|----------------------|
| q0   | any char `c` | q1 | Start new group `c`  |
| q1   | same `c`     | q1 | Extend current group |
| q1   | different / ε | q0 | End group, restart   |

For the full stretchy-word check, the FSA is applied to both `S` and the query word in parallel, comparing each group pair.

---

## 5. State-Transition Table

The table below shows the transitions for the regex `(.)\1*` on the input `"heeellooo"`:

| Step | Current State | Input Char | Same as Group? | Next State | Group So Far |
|------|---------------|------------|----------------|------------|--------------|
| 1    | q0            | `h`        | — (new group)  | q1         | `h`          |
| 2    | q1            | `e`        | No             | q0→q1      | `e`          |
| 3    | q1            | `e`        | Yes            | q1         | `ee`         |
| 4    | q1            | `e`        | Yes            | q1         | `eee`        |
| 5    | q1            | `l`        | No             | q0→q1      | `l`          |
| 6    | q1            | `l`        | Yes            | q1         | `ll`         |
| 7    | q1            | `o`        | No             | q0→q1      | `o`          |
| 8    | q1            | `o`        | Yes            | q1         | `oo`         |
| 9    | q1            | `o`        | Yes            | q1         | `ooo`        |
| 10   | q1            | ε (end)    | —              | Accept     | done         |

Groups produced: `h(1)`, `eee(3)`, `ll(2)`, `ooo(3)`

---

## 6. Deterministic or Non-Deterministic?

**This is a Deterministic Finite Automaton (DFA).**

Here is why:

- **Every state has exactly one transition per input symbol.** In q0, any character moves to q1. In q1, the same character stays in q1, and any different character restarts to q0. There is never a choice between two paths.
- **No ε-transitions.** The automaton never moves without consuming a character (the ε shown in the table above just means end-of-input, not a real transition).
- **No ambiguity.** Given a string, there is exactly one path through the states — the machine's next state is always fully determined by the current state and the current character.

A Non-Deterministic Finite Automaton (NFA) would allow multiple possible next states for the same input, or ε-transitions that move without reading input. This FSA has neither property, so it is deterministic.
