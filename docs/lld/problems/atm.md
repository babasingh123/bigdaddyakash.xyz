# Design an ATM

> **Time:** 40–50 minutes

## Problem

Design the software running on a bank ATM. The system must:

- Accept a **card** and verify the **PIN**.
- Allow the user to **withdraw cash**, **deposit cash**, **check balance**, and **transfer money**.
- Track the ATM's own **cash inventory** and dispense it efficiently (mix of denominations).
- Handle **session lifecycle**: idle → card inserted → authenticated → transacting → ejected.
- Communicate with the **bank backend** to confirm balances and post transactions.
- Recover gracefully from failures (network, dispenser jam, wrong PIN).

## Real-world intuition

An ATM is the textbook **State pattern** problem. Each button press means something completely different depending on whether a card is inserted, the user is authenticated, or a transaction is mid-flight. Without State, you end up with a thicket of `if (currentState == X && hasCard && authenticated)` checks across every input handler. With State, each state knows how to handle each input.

It's also a clean **Command pattern** problem — `Withdrawal`, `Deposit`, and `Transfer` are operations that benefit from a uniform `execute()` interface, with logging and retry handled centrally.

## Key classes

- `ATM` — the machine; holds the current state, card slot, cash dispenser, screen.
- `ATMState` (interface) — `IdleState`, `CardInsertedState`, `AuthenticatedState`, `TransactionState`.
- `Card` — magnetic/chip card with cardholder info and account reference.
- `Account` — bank-side account: balance, type, transaction history.
- `Bank` — backend service abstraction; `verifyPin`, `getBalance`, `debit`, `credit`.
- `Transaction` (abstract, Command) — `execute()`, `rollback()`; concrete `Withdrawal`, `Deposit`, `Transfer`, `BalanceInquiry`.
- `CashDispenser` — physical denomination inventory and dispense algorithm.
- `CashDispensingStrategy` — how to break an amount into available denominations.

## Design patterns used

- **State (ATMState)** — by far the most important pattern. Each state defines what each user input means. `IdleState.insertCard()` transitions to `CardInsertedState`; `CardInsertedState.enterPin()` transitions to `AuthenticatedState`; etc. Eliminates a giant `switch` over the ATM's status.
- **Command (Transaction)** — encapsulates each operation as an object with `execute` and `rollback`. Lets the ATM log every transaction uniformly, retry on transient failure, and roll back atomically when the dispenser jams after debit.
- **Strategy (CashDispensingStrategy)** — multiple algorithms to break $370 into available bills (greedy by denomination, balanced wear across cassettes, prefer-newer-bills). Strategy lets the operations team A/B test without code changes.
- **Singleton (CashDispenser)** — there's exactly one dispenser in the machine; legitimate singleton (hardware resource).

## Class diagram (sketch)

```text
              ┌────────┐
              │  ATM   │
              ├────────┤
              │ -state │──▶ ATMState «interface»
              │ -bank  │           △
              │ -disp. │           ┊
              └────────┘   Idle  CardInserted  Auth  Transaction

              ┌──────────────┐
              │  Transaction │ «abstract, command»
              ├──────────────┤
              │ +execute()   │
              │ +rollback()  │
              └──────△───────┘
        ┌────────┬───┴─────┬────────────┐
   Withdrawal  Deposit  Transfer  BalanceInquiry

   ATM ──▶ CashDispenser ──▶ CashDispensingStrategy «interface»
   ATM ──▶ Bank «service»
```

## Code sketch

```java
interface ATMState {
    void insertCard(ATM atm, Card card);
    void enterPin(ATM atm, String pin);
    void selectTransaction(ATM atm, Transaction t);
    void ejectCard(ATM atm);
}

class IdleState implements ATMState {
    public void insertCard(ATM atm, Card card) {
        atm.setCurrentCard(card);
        atm.setState(new CardInsertedState());
    }
    public void enterPin(ATM atm, String pin)         { atm.display("Insert card first"); }
    public void selectTransaction(ATM atm, Transaction t) { atm.display("Insert card first"); }
    public void ejectCard(ATM atm) { /* nothing to eject */ }
}

class CardInsertedState implements ATMState {
    public void insertCard(ATM atm, Card card)  { atm.display("Card already inserted"); }
    public void enterPin(ATM atm, String pin) {
        if (atm.getBank().verifyPin(atm.getCurrentCard(), pin)) {
            atm.setState(new AuthenticatedState());
        } else if (atm.recordFailedAttempt() >= 3) {
            atm.confiscateCard();
            atm.setState(new IdleState());
        }
    }
    public void selectTransaction(ATM atm, Transaction t) { atm.display("Enter PIN first"); }
    public void ejectCard(ATM atm)                        { atm.eject(); atm.setState(new IdleState()); }
}

class AuthenticatedState implements ATMState {
    public void insertCard(ATM atm, Card c)        { /* ignore */ }
    public void enterPin(ATM atm, String pin)      { /* already auth */ }
    public void selectTransaction(ATM atm, Transaction t) {
        atm.setState(new TransactionState());
        try {
            t.execute(atm);
            atm.logTransaction(t);
        } catch (TransactionException e) {
            t.rollback(atm);
        } finally {
            atm.setState(new AuthenticatedState()); // back to menu
        }
    }
    public void ejectCard(ATM atm) { atm.eject(); atm.setState(new IdleState()); }
}

abstract class Transaction {
    protected final Account account;
    protected final double amount;
    public abstract void execute(ATM atm) throws TransactionException;
    public abstract void rollback(ATM atm);
}

class Withdrawal extends Transaction {
    public void execute(ATM atm) throws TransactionException {
        if (!atm.getBank().debit(account, amount)) throw new InsufficientFundsException();
        try {
            atm.getDispenser().dispense(amount);
        } catch (DispenserException e) {
            atm.getBank().credit(account, amount);  // compensate
            throw new TransactionException(e);
        }
    }
    public void rollback(ATM atm) { /* idempotent rollback */ }
}

interface CashDispensingStrategy {
    Map<Denomination, Integer> breakdown(int amount, Map<Denomination, Integer> available);
}

class GreedyDispensing implements CashDispensingStrategy {
    public Map<Denomination, Integer> breakdown(int amount, Map<Denomination, Integer> avail) {
        Map<Denomination, Integer> out = new EnumMap<>(Denomination.class);
        for (Denomination d : Denomination.values()) {                 // descending order
            int n = Math.min(amount / d.value(), avail.getOrDefault(d, 0));
            if (n > 0) { out.put(d, n); amount -= n * d.value(); }
        }
        if (amount != 0) throw new CannotMakeChangeException();
        return out;
    }
}
```

## Key design decisions

1. **State pattern, not flags** — `boolean cardInserted`, `boolean authenticated`, `int failedAttempts` scattered across methods is the bug factory. The State pattern centralizes "what does this input mean right now?" in one place per state.
2. **Transaction as Command** — `Withdrawal`, `Deposit`, etc. all share `execute`/`rollback`. The ATM doesn't care which it's running — useful for logging, retry, and the compensation pattern on dispenser failures.
3. **Compensating rollback for split commits** — withdrawing money requires (1) debit from bank, (2) dispense bills. If (2) fails after (1) succeeded, we must credit the account back. Encoding this as `rollback()` makes the pattern explicit.
4. **Cash dispensing as a Strategy** — multiple valid algorithms exist (greedy is fastest but can fail; backtracking handles tricky cases). Externalizing lets ops swap without redeploying the ATM firmware.
5. **PIN attempt limit lives on the ATM/card session, not on the bank** — local enforcement prevents an offline attack from racking up retries. The bank also enforces, but the ATM's job is to stop fast.
6. **Singleton dispenser** — the ATM has exactly one cash dispenser. It's hardware; treating it as a singleton is honest (not just convenient).

## State transitions

```text
   ┌──────┐  insertCard   ┌──────────────┐  enterPin (ok)  ┌──────────────┐
   │ Idle │──────────────▶│ CardInserted │────────────────▶│ Authenticated│
   └──────┘               └──────┬───────┘                 └──────┬───────┘
       ▲                         │ 3 failed PINs                  │ select transaction
       │                         ▼                                ▼
       │                ┌────────────────┐                  ┌────────────┐
       └────────────────│ Card confiscated│                  │ Transaction│
        ejectCard        └────────────────┘                  └─────┬──────┘
                                                                   │ done
                                                                   ▼
                                                          (back to Authenticated)
```

## Extensibility

**Add a new transaction type (e.g., bill payment):**

1. `class BillPayment extends Transaction` with its own `execute`/`rollback`.
2. Add to the menu. No edits to states or ATM core.

**Add a new state (Maintenance):**

1. `class MaintenanceState implements ATMState` — rejects all customer inputs, accepts only operator authentication.
2. Trigger transition via a service key/console. No edits to existing states.

**Add a new dispensing algorithm:**

1. `class BalancedWearStrategy implements CashDispensingStrategy` — spread wear across cassettes.
2. Inject. Dispenser logic unchanged.

**Multi-currency support:**

1. `Card` and `Account` learn a currency; `Bank.debit/credit` are per-currency.
2. `CashDispenser` becomes per-currency or holds multiple denominations sets.

## Linked concepts

- [OOP](../fundamentals/oop.md) — abstract `Transaction`, polymorphic state handling.
- [SOLID](../fundamentals/solid.md) — SRP (state, transaction, dispenser are separate), OCP (new states/transactions), DIP (`Bank` is an interface).
- [Design Patterns](../fundamentals/design-patterns.md) — State (canonical example), Command (Transaction), Strategy (Dispensing), Singleton (Dispenser).
