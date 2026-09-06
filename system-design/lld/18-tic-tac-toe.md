# Design Tic-Tac-Toe Game

## 1. Requirements Clarification
**Functional Requirements:**
- N x N board (commonly 3x3).
- Two players (X and O).
- Players take alternating turns.
- Game identifies win or draw conditions efficiently.

**Non-Functional Requirements:**
- Highly optimized move evaluation [O(1) win checking].
- Clean API to plug in an AI player if needed.

> **Interview Tip**: Do NOT use O(N^2) loops to check for a winner after every move. Show the interviewer you know how to track sums.

## 2. Actors & Use Cases
- **Players (Human/AI)**: Make moves.
- **Game Controller**: Manages turns, validates moves, broadcasts results.

## 3. Core Entities & Patterns
- **Board**: Maintains state and validates cell emptiness.
- **Player**: Holds symbol (1 for X, -1 for O).
- **Game**: Orchestrates rules and turn-taking.
- **Design Patterns:**
  - **Strategy Pattern**: *Why?* AI players can have different strategies (Random, Minimax) without altering the `Player` class.
  - **Observer Pattern**: *Why?* UI clients can subscribe to board changes and win events without tightly coupling the game logic to the UI.

## 4. Class Diagram & Architecture

```text
+-------------------+
|      Game         |<>-----> 2 Player
|-------------------|
| - board: Board    |<>-----> 1 Board
| - currentPlayer   |
| + play(row, col)  |
+-------------------+
          | 
          v
+-------------------+       +-----------------------+
|      Board        |       | <<interface>>         |
|-------------------|       |      Strategy         |
| - grid: int[][]   |       |-----------------------|
| - rowSums: int[]  |       | + getMove(board)      |
| - colSums: int[]  |       +-----------------------+
| - diag1, diag2    |                  ^
| + makeMove(r,c,s) |                  |
+-------------------+       +-----------------------+
                            |     MinimaxAI         |
                            +-----------------------+
```

## 5. Deep Dive: Win Checking Optimization
**Naive approach**: After placing a piece, iterate through its row, its column, and the two diagonals to check for a match. **Time Complexity: O(N)** per move.
**Optimal approach**: 
Assign Player 1 the value `+1` and Player 2 the value `-1`.
Maintain arrays:
- `row_sums[N]`
- `col_sums[N]`
- `diagonal_sum`
- `anti_diagonal_sum`

When a player moves at `(r, c)`:
`row_sums[r] += player_value`
`col_sums[c] += player_value`
If `abs(row_sums[r]) == N`, that player has won. 
**Time Complexity: O(1)** per move.

## 6. Full C++ Implementation

```cpp
#include <iostream>
#include <vector>
#include <cmath>
#include <stdexcept>

// SOLID: Single Responsibility Principle (Board only cares about state and win tracking)
class TicTacToeBoard {
private:
    int size;
    std::vector<std::vector<int>> grid;
    std::vector<int> rowSums;
    std::vector<int> colSums;
    int diagSum;
    int antiDiagSum;
    int movesCount;

public:
    TicTacToeBoard(int n) : size(n), grid(n, std::vector<int>(n, 0)), 
                            rowSums(n, 0), colSums(n, 0), 
                            diagSum(0), antiDiagSum(0), movesCount(0) {}

    // Returns 1 if Player 1 wins, -1 if Player 2 wins, 0 otherwise
    int move(int row, int col, int player) {
        if (row < 0 || col < 0 || row >= size || col >= size || grid[row][col] != 0) {
            throw std::invalid_argument("Invalid move.");
        }

        int val = (player == 1) ? 1 : -1;
        grid[row][col] = val;
        movesCount++;

        rowSums[row] += val;
        colSums[col] += val;

        if (row == col) {
            diagSum += val;
        }
        if (row + col == size - 1) {
            antiDiagSum += val;
        }

        if (std::abs(rowSums[row]) == size || 
            std::abs(colSums[col]) == size || 
            std::abs(diagSum) == size || 
            std::abs(antiDiagSum) == size) {
            return player;
        }

        return 0; // No winner yet
    }

    bool isDraw() const {
        return movesCount == size * size;
    }
};

class Game {
private:
    TicTacToeBoard board;
    int currentPlayer; // 1 or 2
    bool gameOver;

public:
    Game(int n) : board(n), currentPlayer(1), gameOver(false) {}

    void playMove(int row, int col) {
        if (gameOver) {
            std::cout << "Game is already over!\n";
            return;
        }

        try {
            int result = board.move(row, col, currentPlayer);
            std::cout << "Player " << currentPlayer << " placed at (" << row << "," << col << ")\n";

            if (result != 0) {
                std::cout << "Player " << currentPlayer << " Wins!\n";
                gameOver = true;
            } else if (board.isDraw()) {
                std::cout << "Game is a Draw!\n";
                gameOver = true;
            } else {
                currentPlayer = (currentPlayer == 1) ? 2 : 1; // Swap turn
            }
        } catch (const std::exception& e) {
            std::cout << e.what() << " Try again.\n";
        }
    }
};

int main() {
    Game game(3);
    game.playMove(0, 0); // P1
    game.playMove(1, 0); // P2
    game.playMove(0, 1); // P1
    game.playMove(1, 1); // P2
    game.playMove(0, 2); // P1 wins!
    return 0;
}
```
