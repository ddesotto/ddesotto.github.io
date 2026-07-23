# The right way to play Wordle

> 

We all know wordle: one word, five letters, 6 attempts. 

I have gotten into the habit of playing with my friends. I am pretty good, I have to admit. But I am pretty competitive. Although I usually come on top, I needed a more reliable way to come on top. 

So lets model it.

## The Game

Wordle is a very simple game. You have 6 guesses to figure out a 5 letter word. Every guess gives you further information on the correct answer. 

A yellow square means the letter in that spot is in the final word. A green square means the letter is both in the word _and_ in the correct position. If the square lights gray, the letter is not in the word.

A bit of _hacking_ tells us there are 2,315 correct answers and 12,972 allowed words. The gap makes it possible to guess words that are never possible. 

So Wordle is a game of guessing. Each guess brings you more information on what the final word is. This links very well with _information theory_.

## Information Theory

Information theory is a framework to study data. I am not going to make this a lecture on information thoery (for that, Claude Shannon's [A mathematical thoery of communication](https://people.math.harvard.edu/~ctm/home/text/others/shannon/entropy/entropy.pdf) is amazing).

Some concepts worth introducing: 

- **Self-information** is how much "surprise" an outcome carries. An outcome with probability $p$ carries

  $$
  I(p) = \log_2 \frac{1}{p} = -\log_2 p \quad \text{bits.}
  $$

  For example, a rare word has a small probability $p$ and hence a high self-information.

  It is a log function, so very independent events (non-related), which are multiplying probabilities, increase the information.

  The units for self-information is the **bit**. Each bit is one halving of the possibilities. i.e. 1/8 = $2^{-3}$ = 3 bits

  ![Each bit of information halves the number of words still in play.](./assets/wordle-bits-halving.svg)

- **Entropy** is the *expected* self-information: how much you learn on     average, before you know the outcome.

  $$
  H = \sum_i p_i \, I(p_i) = \sum_i p_i \log_2 \frac{1}{p_i} \quad \text{bits.}
  $$

  It is a previous guess based on what every outcome could produce. 

  $-H$ is high when therer are many possible equally likely outcomes, nearly zero when one outcome dominates the others.

### The case for Wordle

Every guess gives an answer on what the outcome looks like. Every possible answer pattern is a possiblity. Every tile is one of three colours, so a single guess can return at most $3^5 = 243$ distinct patterns.

Every guess sorts every possible answer into one of these $243$ buckets. Once we get the answer, only one survives.

Over the set of _still can happen_ answers, the probability of pattern $i$ is just the fraction of answers that would produce it:

$$
p_i = \frac{\text{answers that give pattern } i}{\text{answers still possible}}.
$$

Drop that into the definitions from before. The **self-information** of actually seeing pattern $i$ is $-\log_2 p_i$ i.e. how much that specific result narrowed the field. And the **entropy** of the guess is the expected bits it will teach you:

$$
H(\text{guess}) = \sum_{i=1}^{\le 243} p_i \log_2 \frac{1}{p_i}.
$$

So, to recap: 

- a high entropy $H$ splits the buckets into many even buckets.
- a low entropy H give a huge bucket with very high chances of happening (you learn nothing)

The best guess is the one with the highest entopy amongs those available.

![A high-entropy guess splits the words into many small even buckets; a low-entropy one dumps almost everything into a single bucket.](./assets/wordle-bucket-split.svg)

For a possible guess, work out the pattern it *would* produce against every possible answer, tally the buckets, and sum:

```python
from collections import Counter
from math import log2

def entropy(guess, answers):
    buckets = Counter(pattern(guess, answer) for answer in answers)
    total = len(answers)
    return sum((n / total) * log2(total / n) for n in buckets.values())
```

Calling `pattern(guess, answer)` gives the green/yellow/gray key for that pairing. We score very possible word, sort by entropy and the #1 will be the most informative opening. 

We then repeat it on what survives and we will get a mathematically clever way to play the game!!

## Not so quick, greedy

The greedy approach of picking the best answer always, turns out, is not the best. One guess is scored in isolation:

$$
\text{greedy: } \quad \arg\max_{\text{guess}} \; H(\text{guess}).
$$

To play to actually win, you score a guess by the whole game it leads to. The expected number of guesses left once you account for every reply it could get, and the best follow-up to each:

$$
\text{optimal: } \quad \arg\min_{\text{guess}} \; \mathbb{E}\big[\,\text{guesses remaining} \mid \text{guess}\,\big].
$$

This is much more expensive, since you have to search all the possible options. Entropy is a cheap approximation of this, but it is the best answer. 

Two objectives, worth keeping straight:

- **Entropy** answers *"which guess teaches me the most, right now?"* — cheap, greedy, one step.
- **The full search** answers *"which guess wins in the fewest turns, all the way down?"* — expensive, optimal, the whole tree.

The optimal after doing the full tree search comes out to $3.42$ guesses per game. Greedy entropy comes in around $3.6$, barely a fraction of a guess behind. So play harder, not smarter. Unless you are lazy, then definitely play smarter because it is a lot less work.

## Solving
Coming soon...

> I recovered this blog post from my backlog, it is now updated at July 15 2026.

## References

- Claude Shannon, [_A Mathematical Theory of Communication_](https://people.math.harvard.edu/~ctm/home/text/others/shannon/entropy/entropy.pdf) — where self-information and entropy come from
- Alex Selby, [_The best strategies for Wordle_](https://sonorouschocolate.com/notes/index.php/The_best_strategies_for_Wordle) — exhaustive search for the optimal opener (the 3.42 result)
- Laurent Poirrier, [_Wordle-solving state of the art_](https://www.poirrier.ca/notes/wordle-optimal/) — survey comparing greedy-entropy and optimal play
- Bertsimas & Paskov (MIT), [_An Exact and Interpretable Solution to Wordle_](https://mitsloan.mit.edu/ideas-made-to-matter/how-algorithm-solves-wordle) — the optimum re-derived with exact dynamic programming
- Greenberg, [_Effective Wordle Heuristics_](https://arxiv.org/abs/2408.11730) — how close common-word-only play gets to optimal
- Lokshtanov & Subercaseaux, [_Wordle is NP-hard_](https://arxiv.org/abs/2203.16713) — why solving it optimally has no shortcut in general
