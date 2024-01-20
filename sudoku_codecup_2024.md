# The "sudoku" competition on codecup.nl

This is a description of my bot that participated in the 2024 competition of http://codecup.nl.
It reached a third place out of 70 participants.

Sudoku is a 1-player puzzle, but in this competition, the codecup.nl organization converted this to a 2-player game:
Thisgame starts with an empty 9x9 grid. Players take turns placing one digit in a valid cell.
The first player that believes the sudoku has a unique solution claims the victory and wins the game.

The competition started in the summer of 2023, and finished in January 2024, so there was plenty of time to investigate
this game in great depth.

But I did not make good use of this opportunity. I did almost all the work during the summer, then did not touch the code until the week before the finals.
So I am very pleased, and a bit surprised, to end up at third place. The truth is that the winner, Tomek Czajka,
and number two, Tapani Utriainen, were one class better than my bot, while there were a bunch of bots that were about
equally strong as my bot, so I was a bit lucky.

Thanks a lot to the organizers of the competition, and congratulations to Tomek for a well deserved win!

The rest of this document is a description of my bot.

## Sudoku Solver

The first thing we need is a solver that can find solutions for a sudoku puzzle.

There are very advanced solvers on the internet. I studied several such solvers, but did not want to just copy some existing code.
I implemented the algorithm in
"The Really Basic Solver", https://t-dillon.github.io/tdoku/ with a few refinements.

This is a really bad solver for difficult sudokus, but is quite good for finding many solutions in easy sudokus. In this competition, the sudokus
are all very easy, and it is more important to find many solutions quickly. But as we'll
see later, the performance of the solver is not the bottleneck.

On the codecup servers, generating 100 000 solution typically takes 100-500 milliseconds.

## Solution Search

The basic algorithm to find a winning move is:

- Generate all solutions, using the sudoku solver
- Perform a solution search

The most important information maintained for a node in the search tree:
- the number of remaining solutions
- a vector of the indices of the remaining solutions
- for each of the 729 square/value combinations: the "child count" (the number of remaining solutions that have this square/value combination)

sqv here is a "square/value", a number between 0 and 729 (81*9)

The basic algorithm is:

```
search(node, sqv):
  for each index in parent.solutionIndices:
     if solutions[index][sqv.square] != sqv.value: continue
     node.solutionIndices.append(index)
     for each square in solution:
        sqv2 = (square, solution[square])
        node.childCount[sqv2] += 1
  for each sqv2:
     if node.childCount[sqv2] == 1:
        sqv2 is a direct win!
        return "WIN"
  children = list(sqv with node.childCount[sqv] > 1)
  sort children, lowest childCount first
  for sqv2 in children:
     result = search(childNode, sqv2)
     if result == "LOSS":
        sqv2 wins!
        return "WIN"
  return "LOSS"
```

Then there are many optimizations:
- a transposition table is used to detect that we reach the same position from different move orderings or moves.
  Two different moves can lead to the same position in sudoku, so the hash key of a position is based on the squares that have been solved.
- children with child counts 2, 3, 5 always lose, so do not have to be evaluated
- if we test move A, and find that it loses because the opponent can make move B, then there is no need to test move B, because it will lose (opponent will move A and win).
  This optimization works only A's child count <= B's child count, otherwise it can happen that B also wins directly in "our" node.
- if we test move A, and by making this move, square/value combination B is solved, and B has the same child count as A, then A and B will lead to identical positions,
  so there is not need to test B
- many small optimizations to make the updating of nodes (calculating child counts etc) faster

## Game phases

### Opening

First 8 moves: make a random move.

### Middle game

"Small candidate" search. During this phase, we cannot yet generate all solutions, but we are looking for easy wins,
by testing moves that remove most candidates from the current position:

- Make a list of all legal moves
- Sort this list based on the total number of candidates in the position that results after making this move
- Take the move A with least candidates, perform move A, and generate all solutions, up to max 10 000 solutions
- If there were >= 10 000 solutions: make the move with most candidates
- < 10 000 solutions: do a solution search for move A. If we are lucky, A wins
- If A does not win: try to generate 100 000 solutions in the current position.
- If there are < 100 000 positions: do endgame search
- Else: for all moves (starting with the least candidates): do a similar evaluation as we did for move A.
  Stop when we find a winning move, when the resulting solutions >= 10 000, or when allocated time is up

### End game

When there are < 100 000 solutions left: do a solution search as described above.

## Conclusions

Most time I spent on trying to reduce the size of the search tree, thinking, trying, and digging through big log files with dumps of search trees.
I found some things, as described above. Many things did not work out. But overall, I have the nagging feeling that there is something I missed.
