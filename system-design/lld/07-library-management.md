# Design a Library Management System

## Step 1: Requirements Clarification
*   **Actors:** Member, Librarian
*   **Use Cases:** Search books, borrow book, return book, reserve book, fine calculation
*   **Constraints:** A member can borrow max 5 books. [CAP (Consistency, Availability, Partition-tolerance)] Consistency is prioritized for book reservations to prevent double checkouts.

> **Interview Tip:** Differentiate between a Book (the abstract title) and a BookItem (the physical copy with a barcode). This shows attention to detail.

## Step 2: Object Identification
*   **Entities:** `Library`, `Book`, `BookItem`, `Member`, `Reservation`, `Fine`
*   **Enums:** `BookStatus` (AVAILABLE, LOANED, RESERVED, LOST), `AccountStatus` (ACTIVE, CLOSED)

## Step 3: Class Diagram (ASCII)
```text
+---------------+       1..* +---------------+       1..* +---------------+
|    Library    |----------->|     Book      |----------->|   BookItem    |
+---------------+            +---------------+            +---------------+
| -name         |            | -isbn         |            | -barcode      |
| -address      |            | -title        |            | -status       |
+---------------+            | -author       |            +---------------+
                             +---------------+
                                                                ^
                                                                |
+---------------+            +---------------+                  |
|    Member     |<-----------|   BookLending |------------------+
+---------------+   1..*     +---------------+
| -memberId     |            | -creationDate |
| -totalChecked |            | -dueDate      |
+---------------+            | -returnDate   |
                             +---------------+
```

## Step 4: Core APIs / Interfaces
*   `searchByTitle(String title)`
*   `checkoutBookItem(Member member, BookItem item)`
*   `returnBookItem(BookItem item)`
*   `calculateFine(BookLending lending)`

## Step 5: Design Patterns
*   **Observer Pattern:** Notify members when a reserved book is returned. *Why?* Member doesn't need to poll the system; push notification is more efficient.
*   **Strategy Pattern:** Fine calculation (e.g., standard fine, premium member fine). *Why?* Easy to extend fine calculation logic.

## Step 6: Code Implementation (C++)

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <memory>
#include <mutex>

enum class BookStatus { AVAILABLE, LOANED, RESERVED, LOST };

class Member; // Forward declaration

class BookItem {
private:
    std::string barcode;
    BookStatus status;
    std::mutex itemMutex; // [Thread-safety] Ensure thread safety for concurrent checkouts

public:
    BookItem(std::string bc) : barcode(bc), status(BookStatus::AVAILABLE) {}

    std::string getBarcode() const { return barcode; }
    BookStatus getStatus() const { return status; }

    bool checkout() {
        std::lock_guard<std::mutex> lock(itemMutex);
        if (status == BookStatus::AVAILABLE) {
            status = BookStatus::LOANED;
            return true;
        }
        return false;
    }

    void returnItem() {
        std::lock_guard<std::mutex> lock(itemMutex);
        status = BookStatus::AVAILABLE;
    }
};

class Member {
private:
    std::string id;
    int totalBooksCheckedOut;
    const int MAX_BOOKS = 5;

public:
    Member(std::string id) : id(id), totalBooksCheckedOut(0) {}

    bool canCheckout() const { return totalBooksCheckedOut < MAX_BOOKS; }
    void incrementCheckout() { totalBooksCheckedOut++; }
    void decrementCheckout() { totalBooksCheckedOut--; }
    std::string getId() const { return id; }
};

class FineCalculator {
public:
    virtual double calculateFine(int daysOverdue) = 0;
    virtual ~FineCalculator() = default;
};

class StandardFineCalculator : public FineCalculator {
public:
    double calculateFine(int daysOverdue) override {
        return daysOverdue > 0 ? daysOverdue * 1.5 : 0.0;
    }
};

class LibraryManager {
private:
    std::vector<std::shared_ptr<BookItem>> bookItems;
    std::shared_ptr<FineCalculator> fineCalculator;

public:
    LibraryManager(std::shared_ptr<FineCalculator> calc) : fineCalculator(calc) {}

    void addBookItem(std::shared_ptr<BookItem> item) {
        bookItems.push_back(item);
    }

    void checkout(Member& member, std::shared_ptr<BookItem> item) {
        if (!member.canCheckout()) {
            std::cout << "Member reached max checkout limit.\n";
            return;
        }

        if (item->checkout()) {
            member.incrementCheckout();
            std::cout << "Item " << item->getBarcode() << " checked out successfully.\n";
        } else {
            std::cout << "Item " << item->getBarcode() << " is not available.\n";
        }
    }

    void returnBook(Member& member, std::shared_ptr<BookItem> item, int daysOverdue = 0) {
        item->returnItem();
        member.decrementCheckout();
        std::cout << "Item " << item->getBarcode() << " returned.\n";

        if (daysOverdue > 0) {
            double fine = fineCalculator->calculateFine(daysOverdue);
            std::cout << "Fine assessed: $" << fine << "\n";
        }
    }
};

int main() {
    auto calc = std::make_shared<StandardFineCalculator>();
    LibraryManager library(calc);

    auto book = std::make_shared<BookItem>("B123");
    library.addBookItem(book);

    Member alice("M1");

    // Checkout process
    library.checkout(alice, book);
    
    // Attempt double checkout
    library.checkout(alice, book); 

    // Return process
    library.returnBook(alice, book, 2); // 2 days overdue

    return 0;
}
```
