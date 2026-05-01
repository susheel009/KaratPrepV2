# Karat Ladder & Extension Problems (Java) — 10 Problems

Part 2 of the Carrot interview is the "ladder" — you read a multi-class OOP system, fix a failing test, then implement one or two extension methods. 40 minutes, code execution allowed.

Every problem in this document follows the **same structure as the Karat obstacle course / gym membership questions** in the question bank: 3 classes (entity, container with state and a completion flag, aggregator/manager) plus a `Solution` test harness. Bugs are always logical errors in one method (missing filter, wrong filter, wrong field, miscomputed aggregate) — not syntax errors.

## Index

| # | Title | Bug Pattern | Extension Pattern |
|---|---|---|---|
| LADDER-01 | [Library Book Lending](#ladder-01--library-book-lending) | Missing "returned" filter | Most-borrowed book |
| LADDER-02 | [Restaurant Order History](#ladder-02--restaurant-order-history) | Wrong status filter (paid vs all) | Best-selling combo (parallel min/max) |
| LADDER-03 | [Fitness Tracker](#ladder-03--fitness-tracker) | Wrong field aggregated | Average duration per exercise |
| LADDER-04 | [Flight Booking System](#ladder-04--flight-booking-system) | Missing seat-availability filter | Revenue by route |
| LADDER-05 | [Quiz Grading System](#ladder-05--quiz-grading-system) | Counts only one status (gym-membership-style) | Hardest question across attempts |
| LADDER-06 | [Highway Journey Log](#ladder-06--highway-journey-log) | Includes incomplete journeys | Total distance per driver |
| LADDER-07 | [E-commerce Shopping Cart](#ladder-07--e-commerce-shopping-cart) | Discount applied to wrong subset | Best price across carts |
| LADDER-08 | [Music Streaming History](#ladder-08--music-streaming-history) | Missing finished-track filter | Top genre per user |
| LADDER-09 | [Bank Account Ledger](#ladder-09--bank-account-ledger) | Missing posted-status filter | Highest spending category |
| LADDER-10 | [Hotel Reservations + Monte Carlo](#ladder-10--hotel-reservations--monte-carlo) | Missing confirmed filter | Probability simulation (chanceOfPersonalBest analog) |

## How to Run These

Each problem is self-contained. Copy the code into a single `.java` file (e.g., `Solution.java`), compile and run with `javac Solution.java && java -ea Solution` (the `-ea` flag enables `assert` statements). Tests use `assert` — same as the Karat obstacle course code.

For each problem:

1. Set a 35-minute timer.
2. Read the context twice.
3. Read every class aloud, voicing what each does and what conventions are used.
4. Run the code. Read the failing test data — that's the hint.
5. Dry-run the failing case. Find the bug. Fix it.
6. Read the extension request. State the contract aloud. Implement it.
7. If time remains, do Task 3 (or articulate the approach even without coding).

Solutions are in `karat-solutions.md`. Don't peek until you've made an attempt.

---

## LADDER-01 — Library Book Lending

### Context

We are writing software to track a small library's book lending. Members borrow physical books; each loan records the book, the member ID, the borrow date (an integer "day" for simplicity), and whether the book has been returned.

**Definitions:**
- A **`Book`** has an ISBN and a title.
- A **`Loan`** is one borrowing event — book, member ID, day borrowed, day returned (-1 if not returned), and a `returned` flag.
- A **`LoanRegistry`** holds all loans for a particular library and provides statistics.

### Tasks

**Task 1.** Read through the code. The test for `LoanRegistry` is failing because of a bug in one method. Fix it.

**Task 2.** Implement `mostBorrowedBook()` in `LoanRegistry`. It returns the `Book` that has been borrowed the most times (counting all loans, returned or not). If there's a tie, return any of the tied books. If there are no loans, return `null`. Add a test.

**Task 3** (optional). Implement `averageLoanDuration()` — average number of days a book is held, considering only returned loans. Return `0.0` if there are no returned loans.

**[Solution →](karat-solutions.md#ladder-01-solution)**

```java
import java.util.*;

class Book {
    public String isbn;
    public String title;
    
    public Book(String bookIsbn, String bookTitle) {
        isbn = bookIsbn;
        title = bookTitle;
    }
    
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Book)) return false;
        Book b = (Book) o;
        return Objects.equals(b.isbn, this.isbn) && Objects.equals(b.title, this.title);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(isbn, title);
    }
}

class Loan {
    public Book book;
    public int memberId;
    public int dayBorrowed;
    public int dayReturned;  // -1 if not yet returned
    public boolean returned;
    
    public Loan(Book loanBook, int member, int borrowDay) {
        book = loanBook;
        memberId = member;
        dayBorrowed = borrowDay;
        dayReturned = -1;
        returned = false;
    }
    
    public void markReturned(int returnDay) {
        if (returned) {
            throw new IllegalStateException("Loan already returned");
        }
        dayReturned = returnDay;
        returned = true;
    }
    
    public int getDuration() {
        // Returns the number of days the book has been held.
        // If not returned yet, returns -1 (caller must check).
        if (!returned) return -1;
        return dayReturned - dayBorrowed;
    }
}

class LoanRegistry {
    public List<Loan> loans;
    
    public LoanRegistry() {
        loans = new ArrayList<>();
    }
    
    public void addLoan(Loan loan) {
        loans.add(loan);
    }
    
    public int totalLoans() {
        return loans.size();
    }
    
    public int totalActiveLoans() {
        // Returns the count of loans that have NOT been returned yet.
        // BUG: This is producing the wrong count.
        return (int) loans.stream().count();
    }
    
    // TASK 2: implement mostBorrowedBook()
    // public Book mostBorrowedBook() { ... }
    
    // TASK 3 (optional): implement averageLoanDuration()
    // public double averageLoanDuration() { ... }
}

public class Solution {
    public static void main(String[] args) {
        testLoan();
        testRegistry();
    }
    
    public static void testLoan() {
        System.out.println("Running testLoan");
        Book book = new Book("978-0131103627", "The C Programming Language");
        Loan loan = new Loan(book, 42, 10);
        assert !loan.returned : "Loan should not be returned initially";
        assert loan.getDuration() == -1 : "Duration should be -1 before return";
        loan.markReturned(17);
        assert loan.returned : "Loan should be marked returned";
        assert loan.getDuration() == 7 : "Duration should be 7, was " + loan.getDuration();
        try {
            loan.markReturned(20);
            assert false : "marking returned twice should throw";
        } catch (IllegalStateException e) {
            // expected
        }
    }
    
    public static void testRegistry() {
        System.out.println("Running testRegistry");
        Book b1 = new Book("978-0131103627", "The C Programming Language");
        Book b2 = new Book("978-0201633610", "Design Patterns");
        Book b3 = new Book("978-0596007126", "Head First Design Patterns");
        
        LoanRegistry reg = new LoanRegistry();
        
        Loan l1 = new Loan(b1, 1, 1);  l1.markReturned(8);
        Loan l2 = new Loan(b2, 1, 2);  // active
        Loan l3 = new Loan(b1, 2, 3);  l3.markReturned(10);
        Loan l4 = new Loan(b3, 3, 5);  // active
        Loan l5 = new Loan(b1, 4, 6);  // active
        
        reg.addLoan(l1); reg.addLoan(l2); reg.addLoan(l3); reg.addLoan(l4); reg.addLoan(l5);
        
        assert reg.totalLoans() == 5 : "totalLoans should be 5, was " + reg.totalLoans();
        assert reg.totalActiveLoans() == 3 :
            "totalActiveLoans should be 3, was " + reg.totalActiveLoans();
        // Task 2 test (uncomment after implementing):
        // assert reg.mostBorrowedBook().equals(b1) :
        //     "mostBorrowedBook should be b1, was " + reg.mostBorrowedBook().title;
    }
}
```

---

## LADDER-02 — Restaurant Order History

### Context

We are tracking orders at a restaurant. A customer places an `Order` containing several `MenuItem`s. Orders go through statuses: `PENDING`, `PAID`, `CANCELLED`. The kitchen wants statistics about *paid* orders only.

**Definitions:**
- A **`MenuItem`** has a name and a price.
- An **`Order`** has an order ID, a list of items, and a status.
- An **`OrderHistory`** stores all orders for a day and provides aggregate stats.

### Tasks

**Task 1.** A test in `OrderHistory` is failing. Read the code, run it, find and fix the bug.

**Task 2.** Implement `cheapestItemPerCategory()` — given that each `MenuItem` has a `category` field, return a `Map<String, MenuItem>` mapping each category to the cheapest item ever sold in that category, considering items from **paid** orders only. If no items exist for a category, omit it from the map.

**Task 3** (optional). Implement `bestRevenueDay()` — but with the twist that orders span multiple days; given that every `Order` has a `dayPlaced` field, return the day with the highest paid revenue.

**[Solution →](karat-solutions.md#ladder-02-solution)**

```java
import java.util.*;
import java.util.stream.*;

class MenuItem {
    public String name;
    public String category;
    public double price;
    
    public MenuItem(String n, String cat, double p) {
        name = n;
        category = cat;
        price = p;
    }
    
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof MenuItem)) return false;
        MenuItem m = (MenuItem) o;
        return Objects.equals(m.name, name)
            && Objects.equals(m.category, category)
            && Double.compare(m.price, price) == 0;
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(name, category, price);
    }
}

enum OrderStatus { PENDING, PAID, CANCELLED }

class Order {
    public int orderId;
    public List<MenuItem> items;
    public OrderStatus status;
    public int dayPlaced;
    
    public Order(int id, int day) {
        orderId = id;
        dayPlaced = day;
        items = new ArrayList<>();
        status = OrderStatus.PENDING;
    }
    
    public void addItem(MenuItem item) {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException("Cannot add item to non-pending order");
        }
        items.add(item);
    }
    
    public void markPaid() {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException("Order not in PENDING state");
        }
        status = OrderStatus.PAID;
    }
    
    public void cancel() {
        status = OrderStatus.CANCELLED;
    }
    
    public double total() {
        return items.stream().mapToDouble(i -> i.price).sum();
    }
}

class OrderHistory {
    public List<Order> orders;
    
    public OrderHistory() {
        orders = new ArrayList<>();
    }
    
    public void addOrder(Order o) {
        orders.add(o);
    }
    
    public int orderCount() {
        return orders.size();
    }
    
    public double totalRevenue() {
        // Returns total revenue from PAID orders.
        // BUG: revenue is wrong.
        return orders.stream()
            .filter(o -> o.status == OrderStatus.PENDING)
            .mapToDouble(Order::total)
            .sum();
    }
    
    // TASK 2: cheapestItemPerCategory()
    // public Map<String, MenuItem> cheapestItemPerCategory() { ... }
    
    // TASK 3 (optional): bestRevenueDay()
    // public int bestRevenueDay() { ... }
}

public class Solution {
    public static void main(String[] args) {
        testOrder();
        testHistory();
    }
    
    public static void testOrder() {
        System.out.println("Running testOrder");
        Order o = new Order(1, 1);
        o.addItem(new MenuItem("Burger", "main", 12.0));
        o.addItem(new MenuItem("Fries", "side", 4.0));
        assert Math.abs(o.total() - 16.0) < 1e-9 : "total should be 16.0";
        o.markPaid();
        try {
            o.addItem(new MenuItem("Coke", "drink", 3.0));
            assert false : "should not add to paid order";
        } catch (IllegalStateException e) { }
    }
    
    public static void testHistory() {
        System.out.println("Running testHistory");
        OrderHistory hist = new OrderHistory();
        
        Order o1 = new Order(1, 1);
        o1.addItem(new MenuItem("Burger", "main", 12.0));
        o1.markPaid();
        
        Order o2 = new Order(2, 1);
        o2.addItem(new MenuItem("Salad", "main", 10.0));
        // o2 stays PENDING
        
        Order o3 = new Order(3, 1);
        o3.addItem(new MenuItem("Pizza", "main", 15.0));
        o3.markPaid();
        
        Order o4 = new Order(4, 1);
        o4.addItem(new MenuItem("Soup", "main", 8.0));
        o4.cancel();
        
        hist.addOrder(o1); hist.addOrder(o2); hist.addOrder(o3); hist.addOrder(o4);
        
        assert hist.orderCount() == 4 : "orderCount should be 4";
        assert Math.abs(hist.totalRevenue() - 27.0) < 1e-9 :
            "totalRevenue should be 27.0, was " + hist.totalRevenue();
    }
}
```

---

## LADDER-03 — Fitness Tracker

### Context

Tracking workouts. A `Workout` consists of multiple `Exercise` entries. The system keeps a log per user.

### Tasks

**Task 1.** A test in `WorkoutLog` is failing. Find and fix the bug.

**Task 2.** Implement `averageDurationByExercise()` — return a `Map<String, Double>` mapping each exercise name to the average duration (in seconds) across all completed workouts. Skip incomplete workouts.

**Task 3** (optional). Implement `longestStreakDays()` — the longest run of consecutive days the user has at least one completed workout. Each `Workout` has an `int day` field.

**[Solution →](karat-solutions.md#ladder-03-solution)**

```java
import java.util.*;

class Exercise {
    public String name;
    public int durationSeconds;
    public int repetitions;
    
    public Exercise(String exName, int duration, int reps) {
        name = exName;
        durationSeconds = duration;
        repetitions = reps;
    }
}

class Workout {
    public int day;
    public List<Exercise> exercises;
    public boolean completed;
    
    public Workout(int workoutDay) {
        day = workoutDay;
        exercises = new ArrayList<>();
        completed = false;
    }
    
    public void addExercise(Exercise e) {
        if (completed) {
            throw new IllegalStateException("Cannot add to completed workout");
        }
        exercises.add(e);
    }
    
    public void markComplete() {
        completed = true;
    }
    
    public int totalDuration() {
        return exercises.stream().mapToInt(e -> e.durationSeconds).sum();
    }
}

class WorkoutLog {
    public List<Workout> workouts;
    
    public WorkoutLog() {
        workouts = new ArrayList<>();
    }
    
    public void addWorkout(Workout w) {
        workouts.add(w);
    }
    
    public int totalWorkouts() {
        return workouts.size();
    }
    
    public int totalRepetitionsAllExercises() {
        // Sums repetitions across all exercises in completed workouts.
        // BUG: returning the wrong number.
        int total = 0;
        for (Workout w : workouts) {
            if (w.completed) {
                for (Exercise e : w.exercises) {
                    total += e.durationSeconds;
                }
            }
        }
        return total;
    }
    
    // TASK 2: averageDurationByExercise()
    
    // TASK 3 (optional): longestStreakDays()
}

public class Solution {
    public static void main(String[] args) {
        testWorkout();
        testLog();
    }
    
    public static void testWorkout() {
        System.out.println("Running testWorkout");
        Workout w = new Workout(1);
        w.addExercise(new Exercise("squat", 60, 10));
        w.addExercise(new Exercise("plank", 90, 1));
        assert w.totalDuration() == 150 : "totalDuration should be 150";
        w.markComplete();
        try {
            w.addExercise(new Exercise("pushup", 30, 20));
            assert false : "should throw";
        } catch (IllegalStateException e) { }
    }
    
    public static void testLog() {
        System.out.println("Running testLog");
        WorkoutLog log = new WorkoutLog();
        
        Workout w1 = new Workout(1);
        w1.addExercise(new Exercise("squat", 60, 10));
        w1.addExercise(new Exercise("plank", 90, 1));
        w1.markComplete();
        
        Workout w2 = new Workout(2);
        w2.addExercise(new Exercise("squat", 70, 12));
        // w2 NOT complete
        
        Workout w3 = new Workout(3);
        w3.addExercise(new Exercise("pushup", 45, 20));
        w3.addExercise(new Exercise("squat", 65, 15));
        w3.markComplete();
        
        log.addWorkout(w1); log.addWorkout(w2); log.addWorkout(w3);
        
        // 10 + 1 (from w1) + 20 + 15 (from w3) = 46
        assert log.totalRepetitionsAllExercises() == 46 :
            "totalRepetitionsAllExercises should be 46, was " + log.totalRepetitionsAllExercises();
    }
}
```

---

## LADDER-04 — Flight Booking System

### Context

A simple airline booking system. A `Flight` has a fixed seat capacity. Customers make `Reservation`s; some get confirmed, some are waitlisted, some get cancelled. The `BookingSystem` aggregates info across flights.

### Tasks

**Task 1.** A test for `availableSeats()` on `Flight` is failing. Find and fix the bug.

**Task 2.** Implement `revenueByRoute()` in `BookingSystem`: return a `Map<String, Double>` keyed by route (`"JFK->LHR"`) showing total revenue from `CONFIRMED` reservations only.

**Task 3** (optional). Implement `bestSellingClass()` — across the whole system, which travel class (`ECONOMY`, `BUSINESS`, `FIRST`) has generated the most confirmed reservations?

**[Solution →](karat-solutions.md#ladder-04-solution)**

```java
import java.util.*;

enum ReservationStatus { CONFIRMED, WAITLISTED, CANCELLED }
enum TravelClass { ECONOMY, BUSINESS, FIRST }

class Flight {
    public String flightNumber;
    public String origin, destination;
    public int totalSeats;
    public List<Reservation> reservations;
    
    public Flight(String num, String orig, String dest, int seats) {
        flightNumber = num;
        origin = orig;
        destination = dest;
        totalSeats = seats;
        reservations = new ArrayList<>();
    }
    
    public String getRoute() {
        return origin + "->" + destination;
    }
    
    public void addReservation(Reservation r) {
        reservations.add(r);
    }
    
    public int availableSeats() {
        // Returns the number of seats still bookable.
        // BUG: count is wrong.
        return totalSeats - reservations.size();
    }
}

class Reservation {
    public Flight flight;
    public String passengerName;
    public TravelClass travelClass;
    public double pricePaid;
    public ReservationStatus status;
    
    public Reservation(Flight f, String name, TravelClass cls, double price) {
        flight = f;
        passengerName = name;
        travelClass = cls;
        pricePaid = price;
        status = ReservationStatus.CONFIRMED;
    }
    
    public void waitlist() { status = ReservationStatus.WAITLISTED; }
    public void cancel()   { status = ReservationStatus.CANCELLED; }
}

class BookingSystem {
    public List<Flight> flights;
    
    public BookingSystem() {
        flights = new ArrayList<>();
    }
    
    public void addFlight(Flight f) { flights.add(f); }
    
    public int totalConfirmedReservations() {
        return flights.stream()
            .flatMap(f -> f.reservations.stream())
            .filter(r -> r.status == ReservationStatus.CONFIRMED)
            .mapToInt(r -> 1)
            .sum();
    }
    
    // TASK 2: revenueByRoute()
    
    // TASK 3 (optional): bestSellingClass()
}

public class Solution {
    public static void main(String[] args) {
        testFlight();
        testSystem();
    }
    
    public static void testFlight() {
        System.out.println("Running testFlight");
        Flight f = new Flight("AC101", "YYZ", "LHR", 10);
        Reservation r1 = new Reservation(f, "Alice", TravelClass.ECONOMY, 800);
        Reservation r2 = new Reservation(f, "Bob",   TravelClass.ECONOMY, 800);
        Reservation r3 = new Reservation(f, "Carol", TravelClass.ECONOMY, 800);
        r3.cancel();
        Reservation r4 = new Reservation(f, "Dan",   TravelClass.ECONOMY, 800);
        r4.waitlist();
        
        f.addReservation(r1); f.addReservation(r2); f.addReservation(r3); f.addReservation(r4);
        
        // 10 - 2 confirmed (r1, r2) = 8 available
        // (cancelled and waitlisted reservations don't occupy seats)
        assert f.availableSeats() == 8 :
            "availableSeats should be 8, was " + f.availableSeats();
    }
    
    public static void testSystem() {
        System.out.println("Running testSystem");
        BookingSystem sys = new BookingSystem();
        Flight f = new Flight("AC101", "YYZ", "LHR", 10);
        Reservation r1 = new Reservation(f, "Alice", TravelClass.ECONOMY, 800);
        Reservation r2 = new Reservation(f, "Bob",   TravelClass.BUSINESS, 2400);
        f.addReservation(r1); f.addReservation(r2);
        sys.addFlight(f);
        
        assert sys.totalConfirmedReservations() == 2 :
            "should be 2, was " + sys.totalConfirmedReservations();
    }
}
```

---

## LADDER-05 — Quiz Grading System

### Context

A quiz system. A `Quiz` is a sequence of `Question`s. A student takes an `Attempt`; each attempt records which answers were correct/incorrect/skipped per question. This is the closest direct analog to the Karat gym membership question — the bug is identical in spirit (counting only one of two valid statuses).

**Definitions:**
- A `Question` has an ID and a topic.
- An `AnswerStatus` is `CORRECT`, `INCORRECT`, or `SKIPPED`.
- An `Attempt` is one student's pass through the quiz, with a status per question.
- A `QuizSession` stores all attempts for a quiz.

### Tasks

**Task 1.** The `attemptedRate()` method on `QuizSession` is broken. Fix it. (Hint: read the comment carefully — what counts as "attempted"?)

**Task 2.** Implement `hardestQuestion()` in `QuizSession`: across all attempts, return the `Question` with the highest *incorrect rate* (incorrect / (correct+incorrect), excluding skipped). If no question has any non-skipped attempts, return `null`.

**Task 3** (optional). Implement `mostImprovedStudent()` — given each `Attempt` has a `studentId` and a `dayTaken`, find the student whose accuracy improved the most between their first and last attempt.

**[Solution →](karat-solutions.md#ladder-05-solution)**

```java
import java.util.*;

class Question {
    public int id;
    public String topic;
    
    public Question(int qId, String t) {
        id = qId;
        topic = t;
    }
    
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Question)) return false;
        Question q = (Question) o;
        return q.id == this.id && Objects.equals(q.topic, this.topic);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(id, topic);
    }
}

enum AnswerStatus { CORRECT, INCORRECT, SKIPPED }

class Attempt {
    public int studentId;
    public int dayTaken;
    public Map<Question, AnswerStatus> answers;
    
    public Attempt(int student, int day) {
        studentId = student;
        dayTaken = day;
        answers = new LinkedHashMap<>();
    }
    
    public void answer(Question q, AnswerStatus status) {
        answers.put(q, status);
    }
    
    public int correctCount() {
        return (int) answers.values().stream()
            .filter(s -> s == AnswerStatus.CORRECT)
            .count();
    }
}

class QuizSession {
    public List<Question> questions;
    public List<Attempt> attempts;
    
    public QuizSession() {
        questions = new ArrayList<>();
        attempts = new ArrayList<>();
    }
    
    public void addQuestion(Question q) { questions.add(q); }
    public void addAttempt(Attempt a)   { attempts.add(a); }
    
    public double attemptedRate() {
        // Returns: across all answers in all attempts, the fraction of those
        // that were ATTEMPTED (i.e., not skipped). CORRECT + INCORRECT count
        // as attempted. Returns 0.0 if no answers exist.
        // BUG: rate is wrong.
        int total = 0;
        int attempted = 0;
        for (Attempt a : attempts) {
            for (AnswerStatus s : a.answers.values()) {
                total++;
                if (s == AnswerStatus.CORRECT) {
                    attempted++;
                }
            }
        }
        return total == 0 ? 0.0 : (double) attempted / total;
    }
    
    // TASK 2: hardestQuestion()
    
    // TASK 3 (optional): mostImprovedStudent()
}

public class Solution {
    public static void main(String[] args) {
        testAttempt();
        testSession();
    }
    
    public static void testAttempt() {
        System.out.println("Running testAttempt");
        Question q1 = new Question(1, "math");
        Attempt a = new Attempt(100, 1);
        a.answer(q1, AnswerStatus.CORRECT);
        assert a.correctCount() == 1 : "correct should be 1";
    }
    
    public static void testSession() {
        System.out.println("Running testSession");
        QuizSession session = new QuizSession();
        Question q1 = new Question(1, "math");
        Question q2 = new Question(2, "history");
        session.addQuestion(q1);
        session.addQuestion(q2);
        
        Attempt a1 = new Attempt(100, 1);
        a1.answer(q1, AnswerStatus.CORRECT);
        a1.answer(q2, AnswerStatus.SKIPPED);
        
        Attempt a2 = new Attempt(101, 1);
        a2.answer(q1, AnswerStatus.INCORRECT);
        a2.answer(q2, AnswerStatus.CORRECT);
        
        Attempt a3 = new Attempt(102, 1);
        a3.answer(q1, AnswerStatus.CORRECT);
        a3.answer(q2, AnswerStatus.SKIPPED);
        
        session.addAttempt(a1); session.addAttempt(a2); session.addAttempt(a3);
        
        // 6 total answers; 4 are CORRECT or INCORRECT; rate = 4/6 ≈ 0.6667
        assert Math.abs(session.attemptedRate() - 4.0/6.0) < 1e-9 :
            "attemptedRate should be 0.6667, was " + session.attemptedRate();
    }
}
```

---

## LADDER-06 — Highway Journey Log

### Context

A logistics company tracks trucks driving on highways. A `Journey` is one trip a truck takes from origin to destination. As the truck moves, `Segment`s are appended (each segment being a stretch with a recorded distance and time). When the truck reaches the destination, the journey is marked complete; some journeys are abandoned mid-route.

This is the closest analog to the Karat "highway log sheet" extension question.

### Tasks

**Task 1.** A test for `JourneyLog.totalDistanceLogged()` is failing. Fix it. (Hint: the function is supposed to consider only journeys actually completed.)

**Task 2.** Implement `totalDistancePerDriver()` in `JourneyLog`: a `Map<String, Integer>` from driver name to total distance over completed journeys.

**Task 3** (optional). Implement `averageSpeedPerDriver()` — average speed (distance/time) per driver, across completed journeys only. Skip drivers with no completed journeys.

**[Solution →](karat-solutions.md#ladder-06-solution)**

```java
import java.util.*;

class Segment {
    public int distance;       // km
    public int durationMinutes;
    
    public Segment(int dist, int duration) {
        distance = dist;
        durationMinutes = duration;
    }
}

class Journey {
    public String driver;
    public String origin, destination;
    public List<Segment> segments;
    public boolean complete;
    
    public Journey(String driverName, String orig, String dest) {
        driver = driverName;
        origin = orig;
        destination = dest;
        segments = new ArrayList<>();
        complete = false;
    }
    
    public void addSegment(Segment s) {
        if (complete) {
            throw new IllegalStateException("Cannot add segment to completed journey");
        }
        segments.add(s);
    }
    
    public void markComplete() {
        complete = true;
    }
    
    public int totalDistance() {
        return segments.stream().mapToInt(s -> s.distance).sum();
    }
    
    public int totalTime() {
        return segments.stream().mapToInt(s -> s.durationMinutes).sum();
    }
}

class JourneyLog {
    public List<Journey> journeys;
    
    public JourneyLog() {
        journeys = new ArrayList<>();
    }
    
    public void addJourney(Journey j) {
        journeys.add(j);
    }
    
    public int journeyCount() {
        return journeys.size();
    }
    
    public int totalDistanceLogged() {
        // Returns total distance across COMPLETED journeys only.
        // BUG: includes incomplete journeys.
        return journeys.stream()
            .mapToInt(Journey::totalDistance)
            .sum();
    }
    
    // TASK 2: totalDistancePerDriver()
    
    // TASK 3 (optional): averageSpeedPerDriver()
}

public class Solution {
    public static void main(String[] args) {
        testJourney();
        testLog();
    }
    
    public static void testJourney() {
        System.out.println("Running testJourney");
        Journey j = new Journey("Sam", "Toronto", "Montreal");
        j.addSegment(new Segment(100, 60));
        j.addSegment(new Segment(150, 90));
        assert j.totalDistance() == 250 : "totalDistance should be 250";
        assert j.totalTime() == 150 : "totalTime should be 150";
        j.markComplete();
        try {
            j.addSegment(new Segment(50, 30));
            assert false : "should throw";
        } catch (IllegalStateException e) { }
    }
    
    public static void testLog() {
        System.out.println("Running testLog");
        JourneyLog log = new JourneyLog();
        
        Journey j1 = new Journey("Sam", "Toronto", "Montreal");
        j1.addSegment(new Segment(100, 60));
        j1.addSegment(new Segment(150, 90));
        j1.markComplete();
        
        Journey j2 = new Journey("Pat", "Toronto", "Ottawa");
        j2.addSegment(new Segment(50, 30));
        // NOT complete (truck broke down)
        
        Journey j3 = new Journey("Sam", "Montreal", "Quebec City");
        j3.addSegment(new Segment(120, 80));
        j3.addSegment(new Segment(130, 90));
        j3.markComplete();
        
        log.addJourney(j1); log.addJourney(j2); log.addJourney(j3);
        
        // completed: j1 (250) + j3 (250) = 500
        assert log.totalDistanceLogged() == 500 :
            "totalDistanceLogged should be 500, was " + log.totalDistanceLogged();
    }
}
```

---

## LADDER-07 — E-commerce Shopping Cart

### Context

Multiple shoppers, each with their own `Cart`. A `Cart` holds `CartLine`s. Some products are on sale at a discount. The `Marketplace` aggregates across carts.

### Tasks

**Task 1.** A test on `Cart.subtotal()` is failing. Find and fix the bug.

**Task 2.** Implement `bestPricePerProduct()` in `Marketplace`: across all submitted carts (status `CHECKED_OUT`), return a `Map<String, Double>` of product name → lowest price actually paid for that product (price after applying the line's discount). This is the e-commerce analog of `bestOfBests`.

**Task 3** (optional). Implement `mostPopularBundle()` — find the pair of products most frequently appearing together in checked-out carts.

**[Solution →](karat-solutions.md#ladder-07-solution)**

```java
import java.util.*;

class Product {
    public String name;
    public double basePrice;
    
    public Product(String n, double price) {
        name = n;
        basePrice = price;
    }
}

class CartLine {
    public Product product;
    public int quantity;
    public double discountPercent;  // 0.0 to 100.0
    
    public CartLine(Product p, int qty, double discount) {
        product = p;
        quantity = qty;
        discountPercent = discount;
    }
    
    public double effectiveUnitPrice() {
        return product.basePrice * (100.0 - discountPercent) / 100.0;
    }
    
    public double lineTotal() {
        return effectiveUnitPrice() * quantity;
    }
}

enum CartStatus { ACTIVE, CHECKED_OUT, ABANDONED }

class Cart {
    public int cartId;
    public List<CartLine> lines;
    public CartStatus status;
    
    public Cart(int id) {
        cartId = id;
        lines = new ArrayList<>();
        status = CartStatus.ACTIVE;
    }
    
    public void addLine(CartLine line) {
        if (status != CartStatus.ACTIVE) {
            throw new IllegalStateException("Cart is not active");
        }
        lines.add(line);
    }
    
    public void checkout() { status = CartStatus.CHECKED_OUT; }
    public void abandon()  { status = CartStatus.ABANDONED; }
    
    public double subtotal() {
        // Sum of lineTotal() across all lines in this cart.
        // BUG: returning the wrong subtotal.
        double sum = 0;
        for (CartLine line : lines) {
            sum += line.product.basePrice * line.quantity;
        }
        return sum;
    }
}

class Marketplace {
    public List<Cart> carts;
    
    public Marketplace() {
        carts = new ArrayList<>();
    }
    
    public void addCart(Cart c) { carts.add(c); }
    
    // TASK 2: bestPricePerProduct()
    
    // TASK 3 (optional): mostPopularBundle()
}

public class Solution {
    public static void main(String[] args) {
        testCart();
    }
    
    public static void testCart() {
        System.out.println("Running testCart");
        Product p1 = new Product("Book", 20.0);
        Product p2 = new Product("Pen", 2.0);
        
        Cart c = new Cart(1);
        c.addLine(new CartLine(p1, 2, 50.0));   // 2 books at 50% off: 2 * 10 = 20
        c.addLine(new CartLine(p2, 5, 0.0));    // 5 pens at full price: 10
        
        // expected subtotal: 20 + 10 = 30
        assert Math.abs(c.subtotal() - 30.0) < 1e-9 :
            "subtotal should be 30.0, was " + c.subtotal();
    }
}
```

---

## LADDER-08 — Music Streaming History

### Context

A user has a `ListeningHistory`. Each `PlayEvent` records a `Song` and how long the user listened (in seconds). A "completion" only counts if the user listened to at least 90% of the song.

### Tasks

**Task 1.** A test on `ListeningHistory.totalCompletedPlays()` is failing. Find and fix.

**Task 2.** Implement `topGenreByCompletedPlays()` in `ListeningHistory`. Returns the genre name with the most completed plays. Tie: return any. Empty: return `null`.

**Task 3** (optional). Implement `mostSkippedSong()` — the song with the highest skip rate (plays not completed / total plays for that song), considering only songs played at least 3 times.

**[Solution →](karat-solutions.md#ladder-08-solution)**

```java
import java.util.*;

class Song {
    public String title;
    public String artist;
    public String genre;
    public int durationSeconds;
    
    public Song(String t, String a, String g, int dur) {
        title = t;
        artist = a;
        genre = g;
        durationSeconds = dur;
    }
}

class PlayEvent {
    public Song song;
    public int listenedSeconds;
    public int dayPlayed;
    
    public PlayEvent(Song s, int listened, int day) {
        song = s;
        listenedSeconds = listened;
        dayPlayed = day;
    }
    
    public boolean isCompleted() {
        return listenedSeconds >= 0.9 * song.durationSeconds;
    }
}

class ListeningHistory {
    public List<PlayEvent> events;
    
    public ListeningHistory() {
        events = new ArrayList<>();
    }
    
    public void log(PlayEvent e) {
        events.add(e);
    }
    
    public int totalPlays() {
        return events.size();
    }
    
    public int totalCompletedPlays() {
        // Returns count of plays where the user listened to ≥ 90% of the song.
        // BUG: count is wrong.
        return events.size();
    }
    
    // TASK 2: topGenreByCompletedPlays()
    
    // TASK 3 (optional): mostSkippedSong()
}

public class Solution {
    public static void main(String[] args) {
        testHistory();
    }
    
    public static void testHistory() {
        System.out.println("Running testHistory");
        Song s1 = new Song("Wake Me Up", "Avicii",   "EDM",  240);
        Song s2 = new Song("Bohemian",   "Queen",    "Rock", 360);
        Song s3 = new Song("Levitating", "Dua Lipa", "Pop",  200);
        
        ListeningHistory h = new ListeningHistory();
        h.log(new PlayEvent(s1, 230, 1));   // completed (>= 216)
        h.log(new PlayEvent(s1, 100, 2));   // not completed
        h.log(new PlayEvent(s2, 360, 2));   // completed
        h.log(new PlayEvent(s2,  50, 3));   // not completed
        h.log(new PlayEvent(s3, 195, 3));   // completed (>= 180)
        
        assert h.totalPlays() == 5 : "totalPlays should be 5";
        assert h.totalCompletedPlays() == 3 :
            "totalCompletedPlays should be 3, was " + h.totalCompletedPlays();
    }
}
```

---

## LADDER-09 — Bank Account Ledger

### Context

A bank account holds `Transaction`s. Each transaction has a status: `PENDING`, `POSTED`, or `REVERSED`. The current balance is the sum of `POSTED` transactions only. Pending and reversed don't count.

This problem also tests handling of `final` fields and ties to one of the conceptual questions in the bank.

### Tasks

**Task 1.** A test on `Account.balance()` is failing. Find and fix.

**Task 2.** Implement `highestSpendingCategory()` in `AccountLedger` — across all accounts, the spending category (debit only) with the highest total amount considering only POSTED transactions. Each `Transaction` has a `category` field.

**Task 3** (optional). Implement `monthOverMonthChange()` — given each transaction has a `month` field, return a `Map<String, Double>` from category → (this month's spend − last month's spend), looking at the two most recent months in the data.

**[Solution →](karat-solutions.md#ladder-09-solution)**

```java
import java.util.*;

enum TransactionStatus { PENDING, POSTED, REVERSED }
enum TransactionType { DEBIT, CREDIT }

class Transaction {
    public final int id;
    public final double amount;          // always positive; type tells direction
    public final TransactionType type;
    public final String category;        // e.g., "groceries", "rent", "deposit"
    public final int month;              // 1..12 for simplicity
    public TransactionStatus status;
    
    public Transaction(int txId, double amt, TransactionType txType,
                       String cat, int m) {
        id = txId;
        amount = amt;
        type = txType;
        category = cat;
        month = m;
        status = TransactionStatus.PENDING;
    }
    
    public void post()    { status = TransactionStatus.POSTED; }
    public void reverse() { status = TransactionStatus.REVERSED; }
}

class Account {
    public String accountId;
    public List<Transaction> transactions;
    
    public Account(String id) {
        accountId = id;
        transactions = new ArrayList<>();
    }
    
    public void addTransaction(Transaction t) {
        transactions.add(t);
    }
    
    public double balance() {
        // Sum of POSTED transactions: credits add, debits subtract.
        // BUG: balance is wrong.
        double bal = 0;
        for (Transaction t : transactions) {
            if (t.type == TransactionType.CREDIT) {
                bal += t.amount;
            } else {
                bal -= t.amount;
            }
        }
        return bal;
    }
}

class AccountLedger {
    public List<Account> accounts;
    
    public AccountLedger() {
        accounts = new ArrayList<>();
    }
    
    public void addAccount(Account a) { accounts.add(a); }
    
    // TASK 2: highestSpendingCategory()
    
    // TASK 3 (optional): monthOverMonthChange()
}

public class Solution {
    public static void main(String[] args) {
        testAccount();
    }
    
    public static void testAccount() {
        System.out.println("Running testAccount");
        Account a = new Account("ACC-1");
        
        Transaction t1 = new Transaction(1, 1000, TransactionType.CREDIT, "deposit", 1);
        t1.post();
        Transaction t2 = new Transaction(2,  200, TransactionType.DEBIT,  "groceries", 1);
        t2.post();
        Transaction t3 = new Transaction(3,   50, TransactionType.DEBIT,  "coffee",  1);
        // t3 stays PENDING
        Transaction t4 = new Transaction(4,  100, TransactionType.DEBIT,  "rent", 1);
        t4.post();
        t4.reverse();
        
        a.addTransaction(t1); a.addTransaction(t2); a.addTransaction(t3); a.addTransaction(t4);
        
        // Balance: 1000 - 200 = 800 (only POSTED transactions count)
        assert Math.abs(a.balance() - 800) < 1e-9 :
            "balance should be 800, was " + a.balance();
    }
}
```

---

## LADDER-10 — Hotel Reservations + Monte Carlo

### Context

A hotel manages `Booking`s for `Room`s. Each booking has a check-in day, check-out day, and status. The `BookingManager` aggregates.

This problem includes a simulation extension closest in spirit to the obstacle course's `chanceOfPersonalBest`. **Don't attempt Task 3 unless Tasks 1 and 2 are solid and you have time.**

### Tasks

**Task 1.** A test on `BookingManager.confirmedRoomNights()` is failing. Fix it.

**Task 2.** Implement `mostBookedRoom()` in `BookingManager` — the `Room` with the most CONFIRMED bookings. Tie: any. Empty: null.

**Task 3** (optional, simulation). Implement `chanceOfFullOccupancyOnDay(int day, int trials)`. For a given target day:
- Pick `trials` random samples (e.g., 10000) from past confirmed bookings on the same hotel.
- For each trial: randomly assign each room either an occupancy duration drawn uniformly from past confirmed bookings of that room. Check if every room is occupied on `day`.
- Return the fraction of trials where all rooms were occupied on `day`.

This mirrors the `chanceOfPersonalBest` Monte Carlo pattern from the obstacle course problem.

**[Solution →](karat-solutions.md#ladder-10-solution)**

```java
import java.util.*;

enum BookingStatus { CONFIRMED, CANCELLED }

class Room {
    public String roomNumber;
    public String type;  // "single", "double", "suite"
    
    public Room(String num, String t) {
        roomNumber = num;
        type = t;
    }
    
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Room)) return false;
        Room r = (Room) o;
        return Objects.equals(r.roomNumber, this.roomNumber);
    }
    
    @Override
    public int hashCode() {
        return Objects.hashCode(roomNumber);
    }
}

class Booking {
    public Room room;
    public String guestName;
    public int checkInDay;    // day of year
    public int checkOutDay;   // exclusive
    public BookingStatus status;
    
    public Booking(Room r, String guest, int in, int out) {
        room = r;
        guestName = guest;
        checkInDay = in;
        checkOutDay = out;
        status = BookingStatus.CONFIRMED;
    }
    
    public void cancel() { status = BookingStatus.CANCELLED; }
    
    public int nights() {
        return checkOutDay - checkInDay;
    }
    
    public boolean coversDay(int day) {
        return day >= checkInDay && day < checkOutDay;
    }
}

class BookingManager {
    public List<Room> rooms;
    public List<Booking> bookings;
    
    public BookingManager() {
        rooms = new ArrayList<>();
        bookings = new ArrayList<>();
    }
    
    public void addRoom(Room r) { rooms.add(r); }
    public void addBooking(Booking b) { bookings.add(b); }
    
    public int confirmedRoomNights() {
        // Total nights across CONFIRMED bookings.
        // BUG: includes cancelled.
        return bookings.stream()
            .mapToInt(Booking::nights)
            .sum();
    }
    
    // TASK 2: mostBookedRoom()
    
    // TASK 3 (optional): chanceOfFullOccupancyOnDay(int day, int trials)
}

public class Solution {
    public static void main(String[] args) {
        testManager();
    }
    
    public static void testManager() {
        System.out.println("Running testManager");
        Room r1 = new Room("101", "single");
        Room r2 = new Room("102", "double");
        
        BookingManager m = new BookingManager();
        m.addRoom(r1); m.addRoom(r2);
        
        Booking b1 = new Booking(r1, "Alice", 1, 4);    // 3 nights, confirmed
        Booking b2 = new Booking(r1, "Bob",   5, 7);    // 2 nights, cancelled
        b2.cancel();
        Booking b3 = new Booking(r2, "Carol", 1, 5);    // 4 nights, confirmed
        Booking b4 = new Booking(r2, "Dan",   6, 8);    // 2 nights, confirmed
        
        m.addBooking(b1); m.addBooking(b2); m.addBooking(b3); m.addBooking(b4);
        
        // Confirmed: 3 + 4 + 2 = 9
        assert m.confirmedRoomNights() == 9 :
            "confirmedRoomNights should be 9, was " + m.confirmedRoomNights();
    }
}
```

---

## After You Finish

For each ladder problem, after solving:

1. Write a one-liner: "Bug type was \[X\]; key signal in test data was \[Y\]; extension pattern was \[Z\]."
2. Note the time you took. The first one will probably take 50+ minutes — that's normal. By problem 5 you should be hitting 35.
3. If a problem has a Task 3 you skipped, articulate aloud how you'd do it even without coding. That mirrors what to do at minute 38 of the real interview.

Solutions in [`karat-solutions.md`](karat-solutions.md).

---

**← [Back to Master Index](karat-practice-INDEX.md)** | **[Debugging Problems](karat-debugging-problems.md)** | **[Solutions →](karat-solutions.md)**
