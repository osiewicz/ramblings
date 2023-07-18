# Lookup tables in Tree-Sitter
## Speeding up compile times by scaling down ts_lex

The product I work on uses several tree-sitter grammars (via Rust bindings). While looking at our build timings, I have noticed that while most grammars take somewhere between 3 and 8 seconds to build on my machine (M1 Max), `tree-sitter-rust` takes a whooping 30-60 seconds to build.

Each tree-sitter grammar consists of a `scanner.c` file (that - I presume - is hand-written) and auto-generated `parser.c`. Let's confirm that long time is reproducible outside of build environment.

### Timing builds and stuff
I've pulled out a build invokation, which looks somewhat like:
`cc -O3  -c -fPIC -arch arm64 -I ../src -Wall -Wextra -Wno-unused-parameter -Wno-unused-but-set-variable -Wno-trigraphs parser.c`
`cc` on my machine is an alias for `clang`.
Let's time it.
`❯ 28.62s user 0.08s system 99% cpu 28.709 total`

So the issue is real (and the results are similar for `gcc`). I suppose it builds a bit faster than from within `build.rs`, because the machine is otherwise idle - which is obviously not the case when we build this C file from within a build of a big workspace.

Let's pull out a [`-ftime-report`](https://aras-p.info/blog/2019/01/12/Investigating-compile-times-and-Clang-ftime-report/), which should hopefully help us understand what the heck is going on. The output here is trimmed for brevity.

```
cc -O3  -c -fPIC -arch arm64 -I ../src -Wall -Wextra -Wno-unused-parameter -Wno-unused-but-set-variable -Wno-trigraphs parser.c -ftime-report
...
===-------------------------------------------------------------------------===
                      ... Pass execution timing report ...
===-------------------------------------------------------------------------===
  Total Execution Time: 28.2663 seconds (28.2735 wall clock)

   ---User Time---   --System Time--   --User+System--   ---Wall Time---  ---Instr---  --- Name ---
  27.3494 ( 97.5%)   0.0699 ( 32.3%)  27.4192 ( 97.0%)  27.4264 ( 97.0%)  78832431834  CodeGen Prepare
   0.2366 (  0.8%)   0.0006 (  0.3%)   0.2372 (  0.8%)   0.2372 (  0.8%)  2754002559  Simple Register Coalescing
   0.1509 (  0.5%)   0.0805 ( 37.2%)   0.2314 (  0.8%)   0.2314 (  0.8%)  2602662767  AArch64 Instruction Selection
   0.0366 (  0.1%)   0.0604 ( 27.9%)   0.0970 (  0.3%)   0.0970 (  0.3%)  1111385555  AArch64 Assembly Printer
   0.0538 (  0.2%)   0.0000 (  0.0%)   0.0538 (  0.2%)   0.0538 (  0.2%)  887208743  Control Flow Optimizer
   0.0352 (  0.1%)   0.0001 (  0.1%)   0.0353 (  0.1%)   0.0353 (  0.1%)  526434669  Branch Probability Basic Block Placement
   0.0146 (  0.1%)   0.0006 (  0.3%)   0.0152 (  0.1%)   0.0152 (  0.1%)  150762571  Greedy Register Allocator
   0.0133 (  0.0%)   0.0001 (  0.0%)   0.0133 (  0.0%)   0.0133 (  0.0%)  169442985  Machine code sinking
   0.0108 (  0.0%)   0.0006 (  0.3%)   0.0113 (  0.0%)   0.0113 (  0.0%)  106501239  Live Interval Analysis
   0.0106 (  0.0%)   0.0000 (  0.0%)   0.0106 (  0.0%)   0.0106 (  0.0%)   38592607  Two-Address instruction pass
   0.0099 (  0.0%)   0.0002 (  0.1%)   0.0101 (  0.0%)   0.0101 (  0.0%)  154998767  Live Variable Analysis
   0.0092 (  0.0%)   0.0001 (  0.0%)   0.0092 (  0.0%)   0.0092 (  0.0%)   70245763  Machine Common Subexpression Elimination
...
```
So out of these ~30 seconds, we spend over 95% time in "CodeGen Prepare". Now, that's a lead:
- Majority of time is spent in back-end, not front-end. We could probably split parser.c into smaller files to unlock some build parallelism...
- But even then that's unlikely to help. Since majority of time is spent on code generation, it's likely that one of functions contributes to majority of build time. There are only 15 functions in `tree-sitter-rust/src/parser.c`. We would most likely end up with long pole.

### What is ts_lex, anyways?
Out of functions within `parser.c`, `ts_lex` is particularly interesting. It seems to match on Unicode character code and advance lexer's state accordingly.
Each match case looks somewhat like this (the example is artificial to present all possible code flavours and annotated with my comment):
```
case 18:
  // Case #1 (most common): direct comparison with some value.
  if (lookahead == '(') ADVANCE(62);
  if (lookahead == ')') ADVANCE(63);
  if (lookahead == '*') ADVANCE(80);
  if (lookahead == '+') ADVANCE(78);
  if (lookahead == ',') ADVANCE(96);
  if (lookahead == '-') ADVANCE(33);
  if (lookahead == '.') ADVANCE(23);
  if (lookahead == '/') ADVANCE(24);
  if (lookahead == ':') ADVANCE(69);
  if (lookahead == ';') ADVANCE(60);
  if (lookahead == '<') ADVANCE(106);
  if (lookahead == '=') ADVANCE(95);
  if (lookahead == '>') ADVANCE(100);
  if (lookahead == ']') ADVANCE(68);
  if (lookahead == 'r') ADVANCE(157);
  if (lookahead == '{') ADVANCE(64);
  if (lookahead == '|') ADVANCE(117);
  if (lookahead == '}') ADVANCE(65);
  // SKIP macro sets additional variable. We need to keep that in mind.
  if (lookahead == '\t' ||
      lookahead == '\n' ||
      lookahead == '\r' ||
      lookahead == ' ') SKIP(18)
  // Case #2: check against range.
  if (('0' <= lookahead && lookahead <= '9') ||
      ('A' <= lookahead && lookahead <= 'F') ||
      ('a' <= lookahead && lookahead <= 'f')) ADVANCE(46);
  // Case #3: check via a function. It seems like there can be at most one call
  // to such function per switch case.
  if (sym_identifier_character_set_2(lookahead)) ADVANCE(168);
  // Case #4: Negative lookup.
  if (lookahead != 0) ADVANCE(127);
  END_STATE();
case 19:
  ...
```
### Plenty of code, isn't it?
Given how we can have a multitude of these switch cases, it seemed to me that the issues with compile times can be related to the plethora of code that we're dealing with here.
We could instead use a lookup table (mapping a character to state) to reduce the # of if statements from O(n) in characters to check to O(1).
The previous example could be rewritten to use a LUT as follows:
```
case 18:
  // MAX_LUT_VALUE is a max char with set value. We only store ASCII chars in LUT though,
  // so it should never be greater than 127.
  if lookahead < MAX_LUT_VALUE {
    // LUT contains 3 kinds of values:
    // 65535 -> there's no value for given character.
    // [0;32768) -> There's a ADVANCE value for given character.
    // [32768;65535) -> There's a SKIP value for given character.
    // To get the value to pass to SKIP we need to substract 32768 first.
    const static uint16_t lut_18[MAX_LUT_VALUE] = {...};
    uint16_t looked_up_state = lut_18[lookahead];
    if looked_up_state < 32768 {
      ADVANCE(looked_up_state);
    } else if looked_up_state != 65535 {
      SKIP(looked_up_state - 32768)
    }
  }
  // These nodes need to remain, as they can be true for non-ASCII codepoints as well.
  // On the flipside, we skip several checks that cannot be true for non-ascii codepoints,
  // so in a way we speed up UTF-8. Cool.
  if (sym_identifier_character_set_2(lookahead)) ADVANCE(168);
  if (lookahead != 0) ADVANCE(127);
  END_STATE();
```

Does this improve compile times though?
```
❯ 6.11s user 0.04s system 99% cpu 6.156 total
```
Yay! That's over 4x improvement.

How about runtime? Did our change regress the performance somehow?


| ./script/benchmark     | main (89edb2) | lut (9eefac)  |
|------------------------|---------------|---------------|
| Average speed (normal) | 5689 bytes/ms | 5813 bytes/ms |
| Average speed (errors) | 6685 bytes/ms | 6588 bytes/ms |
|                        | [full results](./benchmark_0_base_89edb2ddcaf2928e3197ad6095e1eb1d59bfcc40.txt)              | [full results](./benchmark_1_9eefacf684f181dc132aa89a04f18ffeb763ce6c.txt)              |

Not bad! We got a speed bump in happy cases and slight regression in error parsing speed.

For a good measure, let's also measure binary size differences of `parser.o`.
|                   | main (89edb2) | lut (9eefac) | % difference |
|-------------------|---------------|--------------|--------------|
| bash              | 487432        | 560568       | +14.99%      |
| c                 | 492088        | 530112       | +7.72%       |
| cpp               | 2317200       | 2378512      | +2.65%       |
| embedded-template | 6808          | 10864        | +59.22%      |
| go                | 255960        | 276448       | +7.99%       |
| html              | 13448         | 32112        | +138.43%     |
| java              | 397576        | 424872       | +6.86%       |
| javascript        | 252952        | 278800       | +10.18%      |
| jsdoc             | 17472         | 51936        | +197.07%     |
| json              | 9160          | 15224        | +66.06%      |
| php               | 725024        | 780584       | +7.68%       |
| python            | 417368        | 425488       | +1.95%       |
| ruby              | 2074352       | 2240488      | +7.99%       |
| rust              | 768560        | 780408       | +1.54%       |

There are few tricks we can apply to catch up. In fact, the version we're benchmarking already employs one of them: we only emit LUT for sufficiently large character sets.
It does not make sense to generate full lookup table if there's just handful of comparisons to be made. After all, LUT lookup has it's cost as well. If there are less than 2 states in a given case, we fall back to the old if else chain - it will be more efficient than a LUT anyways.
