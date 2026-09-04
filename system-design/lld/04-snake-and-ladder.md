# Snake & Ladder Game LLD

## 1. Requirements Collection
*   **Actors**: 2-4 Players.
*   **Use Cases**: Roll dice, move player, apply snake (go down) or ladder (go up), detect winner.
*   **Entities**: `Game`, `Board`, `Player`, `Dice`, `Cell`, `Snake`, `Ladder`.

## 2. Terminology & Concepts
*   **Composite Pattern**: Used if we view the Board as a collection of Cells, some of which contain complex entities (Snakes/Ladders).
*   **Strategy Pattern**: Used for dice rolling. Standard dice vs. loaded dice (for testing).

## 3. Class Design & ASCII Diagram
```text
+-------------------+       +-------------------+       +-------------------+
|      Game         |------>|      Board        |------>|      Cell         |
+-------------------+       +-------------------+       +-------------------+
| - players: Queue  |       | - size: int       |       | - id: int         |
| - board: Board    |       | - cells: vector   |       | - jump: Jump*     |
| - dice: Dice      |       +-------------------+       +-------------------+
+-------------------+       | + getCell()       |               |
| + play()          |       +-------------------+               |
+-------------------+                                           v
                                                        +-------------------+
                                                        |      Jump         |
                                                        +-------------------+
                                                        | - start: int      |
                                                        | - end: int        |
                                                        +-------------------+
```

## 4. Design Patterns & SOLID Principles
*   **Strategy Pattern**: `IDice` interface allows injecting a mock dice for deterministic unit testing.
*   **Factory Pattern**: Creating the board with a randomized set of snakes and ladders.

**SOLID Checklist**:
*   **OCP**: `Jump` object handles both Snakes (end < start) and Ladders (end > start). Unified handling means open for extension.

## 5. Full C++ Implementation
```cpp
#include <iostream>
#include <vector>
#include <queue>
#include <string>
#include <cstdlib>

using namespace std;

class Player {
    string name;
    int position;
public:
    Player(string n) : name(n), position(0) {}
    string getName() const { return name; }
    int getPosition() const { return position; }
    void setPosition(int p) { position = p; }
};

struct Jump {
    int start;
    int end;
    Jump(int s, int e) : start(s), end(e) {}
};

class Cell {
    int id;
    Jump* jump;
public:
    Cell(int _id) : id(_id), jump(nullptr) {}
    void setJump(Jump* j) { jump = j; }
    Jump* getJump() const { return jump; }
};

class Board {
    vector<Cell> cells;
    int size;
public:
    Board(int n) : size(n) {
        for(int i=0; i<=size; ++i) cells.push_back(Cell(i));
    }
    
    void addJump(int start, int end) {
        cells[start].setJump(new Jump(start, end));
    }
    
    Cell getCell(int pos) { return cells[pos]; }
    int getSize() const { return size; }
};

class Dice {
    int numDice;
public:
    Dice(int n) : numDice(n) { srand(time(0)); }
    int roll() {
        int total = 0;
        for(int i=0; i<numDice; ++i) {
            total += (rand() % 6) + 1;
        }
        return total;
    }
};

class Game {
    Board board;
    Dice dice;
    queue<Player*> players;
    
public:
    Game(Board b, Dice d) : board(b), dice(d) {}
    
    void addPlayer(Player* p) { players.push(p); }
    
    void play() {
        while (players.size() > 1) {
            Player* currPlayer = players.front();
            players.pop();
            
            int roll = dice.roll();
            int newPos = currPlayer->getPosition() + roll;
            
            if (newPos > board.getSize()) {
                players.push(currPlayer); // Skip turn if overshoot
                continue;
            }
            
            Jump* jump = board.getCell(newPos).getJump();
            if (jump) {
                cout << currPlayer->getName() << " hit a " 
                     << (jump->end > jump->start ? "Ladder!" : "Snake!") << "\n";
                newPos = jump->end;
            }
            
            currPlayer->setPosition(newPos);
            cout << currPlayer->getName() << " moved to " << newPos << "\n";
            
            if (newPos == board.getSize()) {
                cout << currPlayer->getName() << " WINS!\n";
            } else {
                players.push(currPlayer);
            }
        }
    }
};

int main() {
    Board board(100);
    // Ladders
    board.addJump(2, 38);
    board.addJump(15, 26);
    // Snakes
    board.addJump(98, 20);
    board.addJump(50, 5);
    
    Dice dice(1); // 1 die
    Game game(board, dice);
    
    Player p1("Alice");
    Player p2("Bob");
    game.addPlayer(&p1);
    game.addPlayer(&p2);
    
    game.play();
    return 0;
}
```

## 6. Interview Tips & Follow-up Questions
*   **Q: How to handle 3 consecutive 6s rules?**
    *   A: Add logic in the `Game` loop. If `roll == 6`, player gets another turn. Track a consecutive 6s counter. If it hits 3, reset position to previous state and end turn.
*   **Tip**: Unifying Snakes and Ladders under a single `Jump` struct simplifies the Board traversal logic considerably. This is a key design choice interviewers look for.

