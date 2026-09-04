# Chess Game LLD

## 1. Requirements Collection
*   **Actors**: Two Players.
*   **Use Cases**: Move piece, validate move rules, detect Check/Checkmate, En Passant, Castling.
*   **Entities**: `Game`, `Board`, `Cell`, `Piece`, `Player`, `Move`.

## 2. Terminology & Concepts
*   **En Passant**: Special pawn capture rule.
*   **Castling**: King and Rook move simultaneously under specific conditions.
*   **Check**: King is under attack.
*   **Checkmate**: King is under attack and has no legal moves.

## 3. Class Design & ASCII Diagram
```text
+-------------------+       +-------------------+
|      Game         |------>|      Board        |
+-------------------+       +-------------------+
| - board: Board    |       | - cells: Cell[][] |
| - players: P[]    |       +-------------------+
| - turn: Player    |               |
+-------------------+               v
| + makeMove(Move)  |       +-------------------+
+-------------------+       |      Cell         |
        |                   +-------------------+
        | uses              | - x, y: int       |
        v                   | - piece: Piece*   |
+-------------------+       +-------------------+
|      Move         |               |
+-------------------+               v
| - start: Cell     |       +-------------------+
| - end: Cell       |       |     Piece         |
| - pieceMoved: P   |       +-------------------+
+-------------------+       | - color: Color    |
                            +-------------------+
                            | + isValidMove()   |
                            +-------------------+
                                    ^
                                    | inherits
                    +-------------------------------+
                    |                               |
              +-----------+                   +-----------+
              |   King    |                   |  Knight   | ...
              +-----------+                   +-----------+
```

## 4. Design Patterns & SOLID Principles
*   **Strategy / Polymorphism**: `Piece` is an abstract base class. Each specific piece (Knight, Bishop) implements its own `isValidMove()` logic.
*   **Command Pattern**: The `Move` object acts as a command. This makes implementing "Undo" trivial by storing a stack of `Move` objects.
*   **Observer Pattern**: To notify UIs or loggers when the game state changes (e.g., Checkmate).

**SOLID Checklist**:
*   **OCP**: Adding a new piece variant (like in fairy chess) only requires extending `Piece`.
*   **LSP (Liskov Substitution)**: Any `Piece*` can be evaluated for movement without knowing its specific type.

## 5. Full C++ Implementation (Focus on Movement & Validation)
```cpp
#include <iostream>
#include <vector>
#include <cmath>
#include <memory>

using namespace std;

enum class Color { WHITE, BLACK };

class Board; // Forward declaration

class Piece {
protected:
    Color color;
public:
    Piece(Color c) : color(c) {}
    virtual bool isValidMove(Board& board, int startX, int startY, int endX, int endY) = 0;
    Color getColor() const { return color; }
    virtual ~Piece() = default;
};

class Cell {
    int x, y;
    Piece* piece;
public:
    Cell(int _x, int _y) : x(_x), y(_y), piece(nullptr) {}
    void setPiece(Piece* p) { piece = p; }
    Piece* getPiece() const { return piece; }
};

class Board {
    vector<vector<Cell>> cells;
public:
    Board() {
        for(int i=0; i<8; ++i) {
            vector<Cell> row;
            for(int j=0; j<8; ++j) row.push_back(Cell(i, j));
            cells.push_back(row);
        }
    }
    Cell& getCell(int x, int y) { return cells[x][y]; }
};

class Knight : public Piece {
public:
    Knight(Color c) : Piece(c) {}
    bool isValidMove(Board& board, int startX, int startY, int endX, int endY) override {
        // Validation logic for L-shape
        int dx = abs(startX - endX);
        int dy = abs(startY - endY);
        if ((dx == 2 && dy == 1) || (dx == 1 && dy == 2)) {
            Piece* destPiece = board.getCell(endX, endY).getPiece();
            // Valid if empty or enemy piece
            if (!destPiece || destPiece->getColor() != this->color) {
                return true;
            }
        }
        return false;
    }
};

class Move {
    int startX, startY;
    int endX, endY;
public:
    Move(int sx, int sy, int ex, int ey) : startX(sx), startY(sy), endX(ex), endY(ey) {}
    int getStartX() const { return startX; }
    int getStartY() const { return startY; }
    int getEndX() const { return endX; }
    int getEndY() const { return endY; }
};

class Game {
    Board board;
    Color currentTurn;
public:
    Game() : currentTurn(Color::WHITE) {
        // Simplified setup
        board.getCell(0, 1).setPiece(new Knight(Color::WHITE));
    }
    
    bool makeMove(Move move) {
        Cell& start = board.getCell(move.getStartX(), move.getStartY());
        Cell& end = board.getCell(move.getEndX(), move.getEndY());
        
        Piece* p = start.getPiece();
        if (!p || p->getColor() != currentTurn) {
            cout << "Invalid piece selection.\n";
            return false;
        }
        
        if (p->isValidMove(board, move.getStartX(), move.getStartY(), move.getEndX(), move.getEndY())) {
            // Check for Check logic would go here before finalizing move
            end.setPiece(p);
            start.setPiece(nullptr);
            currentTurn = (currentTurn == Color::WHITE) ? Color::BLACK : Color::WHITE;
            cout << "Move successful.\n";
            return true;
        }
        cout << "Invalid move for this piece.\n";
        return false;
    }
};

int main() {
    Game game;
    // Attempt valid Knight move
    game.makeMove(Move(0, 1, 2, 2));
    return 0;
}
```

## 6. Interview Tips & Follow-up Questions
*   **Q: How do you detect Check/Checkmate?**
    *   A: **Check**: After every move, iterate through all opponent pieces. If any piece's `isValidMove` targets our King's current cell, we are in Check.
    *   A: **Checkmate**: If in Check, generate all possible moves for all our pieces. Apply each move temporarily. If *none* of those moves result in getting out of Check, it's Checkmate.
*   **Tip**: The `isValidMove` in `Piece` checks logical geometry (e.g., L-shape). The `Game` class must check state-level rules (e.g., "Does making this move put my own king in check?"). Don't mix them up.

