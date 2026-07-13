# The math in enclose.horse

> What a game about trapping horses can teach you about graphs and optimization

I've lately been hooked to this simple game called [enclose.horse](https://enclose.horse/). I figured: since i liked winning at it, why not solve it?

## The game

The rules are simple:

- You have $k$ walls;
- you can place them on grass tiles;
- the horse moves up, down, left, and right, never diagonally;
- existing obstacles like water and rocks already block movement;
- if the horse reaches the edge of the grid, it escapes;
- your score comes from the tiles trapped inside.

![An example enclose.horse map](./assets/enclose-horse-map.png)

To make it more interesting, there are a few extras: 
- tiles with a cherry give you a `+3` point bonus;
- tiles with a golden apple give you a `+10` point bonus;
- tiles with a bee swarm apply a `-5` penalty;
- portals can teleport the horse from one portal tile to the other.

The game reduces to:

> Block a set of cells so the horse's region is cut off from the outside, while maximizing the score of that region.

## The model

Lets work up from a simplified version of the game.

### Trapping the horse

A good place to start is with the first part of the problem: _block a set of cells so the horse's region is cut off from the outside_. Forget scoring, forget the wall limit, and ask the simplest possible question:

> What is the _fewest_ walls that trap the horse at all?

Since we are trying to block all paths, this is a paths problem — and paths problems ask for graphs.

- every accessible cell is a vertex;
- every legal horse move is an edge;
- every blocked cell is removed from the graph;
- and the entire outside world is one extra special vertex $o$, joined to every boundary cell the horse could escape through (this is to account for the horse being able to move outside the map)

![Grid cells become vertices, legal moves become edges, and the outside is one vertex.](./assets/enclose-horse-grid-to-graph.svg)

"Trap the horse" means we have to separate the horse vertex $h$ from the outside vertex $o$ by deleting cells. Since walls sit *on cells*, the walls we place form a **vertex cut**: a set of vertices whose removal leaves no path from $h$ to $o$. The fewest walls is therefore the **minimum vertex cut**. This is solveable via [Menger's theorem](https://en.wikipedia.org/wiki/Menger%27s_theorem): the minimum number of cells we must delete to separate $h$ from $o$ equals the maximum number of *vertex-disjoint* (or the number of escape points) paths the horse could take to escape.

The classic max-flow / min-cut theorem cuts *edges*, not vertices, so we can't apply it directly. The solution is **node splitting**: replace each cell $v$ with an in/out pair $v_\text{in} \to v_\text{out}$ joined by an edge whose capacity is the wall cost, and give the original moves infinite capacity. A minimum *edge* cut in the split graph can only sever those internal edges, so it corresponds exactly to a minimum *vertex* cut in the original description. 

So we build the split graph, run max-flow, and the minimum cut tells us the fewest walls required in polynomial time. Good, but sadly not the game we are playing.

### Maximizing the score

Trapping the horse is solved, but that was never really the game. You have $k$ walls, and your score is whatever you manage to seal off. So the real goal isn't the *cheapest* trap; it's the most *valuable* one.

Now we add the $k$ wall constraint back.

Unfortunately, the moment you have a budget, the question changes from _if?_ to _where?_. Without a budget you always want the fewest walls (the minimum cut). But with $k$ walls to spend, the cheapest trap stops being the trap you *want*: you can afford to spend more, and spending more can wrap a bigger, more valuable field. So the min-cut answer is the wrong target.

To see why a longer fence is even a coherent idea, you can think of the trapping wall set as a **loop**. Any set of walls that seals the horse in forms a closed fence around it, and in the plane a closed curve splits everything into an inside and an outside (fancily, the [Jordan Curve Theorem](https://mathworld.wolfram.com/JordanCurveTheorem.html)). The horse is stuck on the inside, and your score is whatever that inside contains.

So the real objective is:

$$
\max \sum_{v \in R_h(S)} p_v
\qquad \text{subject to} \qquad
|S| \le k
$$

i.e. maximize the value of the horse's sealed-off region $R_h(S)$, using at most $k$ walls. To keep things simple, let's start by treating every tile as worth one point — the goal is then just to enclose *as many tiles as possible*. We'll bring the real scores back when we get to cherries.

![The cheapest trap hugs the horse; the budget wants the largest valuable region.](./assets/enclose-horse-fence-vs-field.svg)

We switch from optimizing the loop to optimizing the area within. The value of a wall is now decided by the entire region left behind, not by the wall itself. We have stopped optimizing the fence and started optimizing the field.

The requirements are now to talk about _which side each tile ended up on_. For example, we can use two binary variables per tile:

- $W_v \in \{0,1\}$ — do we place a wall on $v$? (your actual move)
- $E_v \in \{0,1\}$ — is $v$ an escape route, i.e. still connected to the outside?

Maximizing enclosed tiles is the same as minimizing escape ones:

$$
\begin{aligned}
\text{minimize} \quad & \sum_v E_v \\
\text{subject to} \quad & \sum_v W_v \le k, \\
& E_h = 0,\quad W_h = 0, \\
& E_v = 1 - W_v && \text{for border tiles,} \\
& E_u \ge E_v - W_u && \text{for every ordered adjacent pair } (u, v).
\end{aligned}
$$

That last constraint is the whole trick. We need a linear way to say _is $u$ escapable?_. You can think of "escapability" as a disease creeping in from the outside, and we have to stop it from reaching the horse. A wall is the way to do that. Forcing $E_h = 0$ means: no unwalled path may leak from the horse to the border. It's the same idea as for the flow, but flattened into one inequality per edge.

```text
minimize   sum(E[v] for v in tiles)
s.t.       sum(W[v] for v in tiles) <= k
           E[horse] = 0,  W[horse] = 0
           E[v] = 1 - W[v]              for v on the border
           E[u] >= E[v] - W[u]          for every ordered adjacent pair (u, v)
           W[v], E[v] in {0, 1}
```

Since the variables are integers, this is an _integer_ linear program. 

### Cherries, golden apples, bees and portals

Now, if you have ever designed a game, you know how important it is to add some extras to make it more interesting. [enclose.horse](https://enclose.horse/) uses special tiles that change the gameplay (and hence the model) enough to frustrate you a bit longer. This game introduces the following:

- **Cherries (+3)** and **Golden Apples (+10)** are just **weights in the objective function**. Swap the sum $\sum_v E_v$ for a weighted $\sum_v p_v E_v$ and you are done.

- **Portals** are just **extra edges**. The horse teleports between a paired set of portals, so in the graph you draw one edge joining them; escapability flows across that edge exactly the way the horse does. Intuitively, if you play the game with no model, portals are complex to grasp, because visualizing the connectivity is complex. Luckily, math does not care about this complexity, and this is very easy to model.

![A portal is just another edge, the escapability constraint does not care how far it reaches.](./assets/enclose-horse-portals.svg)

**Bees (−5)** are the one tile you would rather leave *outside* your fence, and that single negative number breaks an assumption we never stated: that every tile we *can* trap is worth trapping. Minimizing $\sum_v p_v E_v$ only behaves when every $p_v$ is positive.

If we flip one score negative the incentive inverts. The solver *wants* $E_\text{bee} = 1$, because $-5 \cdot E_\text{bee}$ drops the objective by five. So it labels a fully walled-in bee an *escape route*, even though it is sealed on all four sides. Nothing forbids this, because every constraint we wrote only ever pushes escapability *up* ($E_u \ge E_v - W_u$) and never pins it *down*.

The obvious patch is to add the reverse: a tile may be escapable only if it is unwalled and has an escapable neighbor.

$$
E_v \le 1 - W_v, \qquad E_v \le \sum_{u \sim v} E_u
$$

However, this is still not enough. The solver still *wants* $E_\text{bee} = 1$, so it only needs some way to satisfy the new "must have an escapable neighbor" rule — and it can fabricate one. Picture a ring of tiles sitting in the enclosed region, each pointing at the next, all labelled "escapable". Every tile in the ring has an escapable neighbor (the next one along), so the constraints hold, yet the ring touches the outside *nowhere*.

To kill this "invisible" ring we have to force escapability to actually *reach the boundary*. Give every escapable tile one unit of "proof of escape" that it must ship out to the outside vertex $o$, travelling only through unwalled tiles:

$$
\sum_{u} f_{v \to u} - \sum_{u} f_{u \to v} = E_v, \qquad f_{u \to v} \le |V|\,(1 - W_u)
$$

Now a walled bee is trapped on paper too: every path from it to $o$ crosses a wall, so its unit of proof has nowhere to go and $E_\text{bee} = 1$ turns *infeasible* (impossible).

## Solving

I implemented both models in Python with [Google OR-Tools](https://developers.google.com/optimization).

The code is on GitHub: [ddesotto/enclose-horse-solver](https://github.com/ddesotto/enclose-horse-solver). It's a small CLI you can point at the puzzle url:

```sh
uv run enclose-horse https://enclose.horse/play/2026-07-12 --mode solve
```

## References

- [enclose.horse: How to Play](https://enclose.horse/)
- [ddesotto/enclose-horse-solver](https://github.com/ddesotto/enclose-horse-solver) — the solver described in this post
- [dynomight, _Integer programming easily encloses horse_](https://dynomight.substack.com/p/horse)
- [enclose.horse discussion on Tildes](https://tildes.net/~games/1s35/browser_puzzle_game_enclose_horse)
- Ahuja, Magnanti, and Orlin, _Network Flows: Theory, Algorithms, and Applications_
