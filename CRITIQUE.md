# RoomReserve Critique

Fill in each section. One section per milestone. Keep it short and specific. Point at
files and methods, not adjectives.

---

## Milestone 1: The design as it is

Describe the system as the code actually builds it.

**Data model.** What is a booking, in the code? What types hold it, and what has to stay
in agreement for a booking to make sense?
- InMemoryStore.java line 16
- A booking is represented by an array containing a room, date, start/end time, and user, represented by string, string, long, long, string, respectively. The room and date are stored together as the key in slotsByRoomDate, while the user is stored separately in bookerBySlot. These structures must stay synchronized: the booking must exist in both maps with the correct room, date, times, and user.


**Operations.** What can a caller do, and what goes in and out?
In 'RequestHandler.java', a caller can 
1) createBooking(room, date, start, end, user) parses the time strings, checks that the interval does not overlap an existing booking, and stores it. It returns a success or error string.
2) cancelBooking(room, date, start, end) parses the times and removes the exact matching interval. It returns a success or error string.
3) rescheduleBooking(room, date, oldStart, oldEnd, newStart, newEnd) finds the existing booking, removes it, and adds it at the new time while preserving the user. It returns a success or error string. It does not check the new interval for overlap or business-hour rules.
4) listBookings(room, date) returns a formatted string containing all bookings for that room and date, including times and users, or a message if none exist.

**Structure.** What classes exist, what does each own, and who holds a reference to whom?
1) ReservationApp: serves as a demo entry point, creating own script of requests (to simulate), holds reference to RequestHandler
2) RequestHandler: owns, parses, perfoms some validation for the requests, holds reference to InMemoryStore
3) BookingPolicy: owns the rules and methods for business hours, maxiumum length, and overlap vadliation, holds reference to none of them. 
4) InMemoryStore: owns the in-memory booking data using slotsByRoomDate and bookerBySlot. It provides methods to add, remove, find, and list bookings.

**The no-double-booking invariant.** Where is it enforced? Name every place a check
happens, say what each one actually checks, and trace one reschedule request through the
code from the entry point to storage.
- The invariant is partially enforced in two places:
1) RequestHandler.createBooking, lines 30–36, loops through existing bookings and rejects overlapping intervals.
2) InMemoryStore.addSlot, lines 28–40, rejects an exact duplicate start/end interval.
- BookingPolicy.validate also contains an overlap check, but no code calls it, so it does not enforce the invariant in practice. scheduleBooking does not check for overlap at all and ignores whether adding the new slot fails.
Trace (from ReservationApp): 
- ReservationApp calls handler.rescheduleBooking(...) ---> RequestHandler parses the old/new times --> calls store.bookerFor(...) --> calls store.removeSlot(...) --> calls store.addSlot(...), which writes the new interval into slotsByRoomDate and the user into bookerBySlot


## Milestone 2: Two design problems

Two problems. For each one, fill in all three parts.

### Problem 1

**The problem.** Name it, using the vocabulary from lecture (milestone 2 in the
handout names the three).
Misplaced Responsbility

**Where in the code.** File and method.
RequestHandler.java with the createBooking and rescheduleBooking methods doing validation stuff

**What it makes expensive.** A concrete future change, or something that already goes
wrong today. What breaks first?
RequestHandler directly checks endMinutes <= startMinutes and overlaps, even though BookingPolicy owns booking rules and has validate. For instance, a future function is to do 'Per-building business hours instead of one 08:00 to 20:00 window.' In this current design, we are delegating a check that should be reserved for BookingPolicy to ensure this, not createBooking or rescheduleBooking.

### Problem 2

**The problem.**
Representational gap

**Where in the code.**
InMemoryStore.java in bookerBySlot

**What it makes expensive.**
Adding booking data such as an ID, purpose, or creation time requires changing multiple structures and keeping them synchronized. This representation can also produce inconsistent state if one structure changes without the other.
---

## Milestone 3: Two alternative decompositions

Two different ways to carve up this system. A different split of responsibility, not a
list of local code fixes. Read the handout's appendix before writing this section.

### Alternative A

**The decomposition.** What are the pieces, what does each own, and where do the rules
live?
Keep RequestHandler responsible for reading request strings and creating response messages. Add a ProcessRequest to be responsible for creating, cancelling, and rescheduling bookings. ProcessRequest always asks BookingPolicy whether an operation is valid before changing the store. Replace the separate time arrays and user map with one Booking object containig the room, data, times, and user. InMemoryStore stores and retrieves these Booking objects, 

**One tradeoff.** Something this option actually costs. "No real downside" is not a
tradeoff.
This creates more classes and makes simple requests pass through more code, so there may be difficulty tracing there for a smaller applicatoin. 

### Alternative B

**The decomposition.**
Create a RoomSchedule class for each room and date. It stores that schedule's bookings and decides whether a new or rescheduled booking follows the rules. InMemoryStore finds the correct RoomSchedule and asks it to create, cancel, or reschedule a booking. RequestHandler only reads the request and writes the response. 

**One tradeoff.**
RoomSchedule may become large because it stores bookings and enforces all the rules. If different rooms later have different rules, this
design may need more changes.

### Preference

Which one, and under what conditions? Say what the choice depends on, and what would
make you pick the other one instead.
I prefer Alternative A if the application will support 'per-building business hours instead of one 08:00 to 20:00 window', because all booking operations have one place that coordinates the rules. I prefer Alternative B if the application stays small, because each room’s schedule directly protects itself from invalid bookings.