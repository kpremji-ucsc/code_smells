# Large Class

This presentation is about the code smell **Large Class** with examples in **C#**, **TypeScript**, and **Python**.

## What Is It?

A **Large Class** is a class that tries to do too much: too many fields, methods, and responsibilities. Funny enough this is also known as a **God Object** because it knows and controls everything.

A quick test: if describing the class takes the word **"and"** ("it moves the player *and* tracks health *and* updates the UI *and* saves the game"), it probably has too many jobs.

### How It Arises

Classes rarely start large. A `Player` class begins with movement, and then it's just *easier* to add health there, then inventory, then UI updates, because the class is already open and has access to everything. Although each addition is quite small and non-trivial, it eventually adds up to be a giant mess that nobody wants to deal with, so sectioning off early is important.

### Why It's a Problem

- **Hard to understand:** you have to read hundreds of lines to change one behavior.
- **Fragile:** unrelated features share state, so a change to inventory can break health.
- **Hard to reuse:** an enemy needs health too, but it can't borrow it without dragging along player input and UI.
- **Merge conflicts:** everyone on the team edits the same file.

## Prevention and Refactoring

**Prevention**
- Follow the **Single Responsibility Principle**: each class should have one primary purpose
- Do small **litter-pickup refactorings** (Fowler, Ch. 2) whenever you touch the class, before it grows out of control.

**Refactoring** (from [Fowler's catalog](https://refactoring.com/catalog/))
1. [Extract Class](https://refactoring.com/catalog/extractClass.html): move a related group of fields and methods into a new class; extract one responsibility at a time, and test after each step.
2. [Move Function](https://refactoring.com/catalog/moveFunction.html) / [Move Field](https://refactoring.com/catalog/moveField.html): relocate members to the class that uses them most.



## Examples

### C#: A Unity Player God Class

**Before:** one class handles movement, health, inventory, and UI.

```csharp
public class Player : MonoBehaviour
{
    public float speed = 5f;
    public int maxHealth = 100;
    public Text healthText;

    Rigidbody2D rb;
    int health;
    List<string> items = new();

    void Awake()
    {
        rb = GetComponent<Rigidbody2D>();
        health = maxHealth;
    }

    void Update()
    {
        var input = new Vector2(Input.GetAxis("Horizontal"), Input.GetAxis("Vertical"));
        rb.velocity = input * speed;
    }

    public void TakeDamage(int amount)
    {
        health -= amount;
        healthText.text = $"HP: {health}";
        if (health <= 0) Destroy(gameObject);
    }

    public void AddItem(string item) => items.Add(item);
}
```

Enemies need health too, but they'd have to copy this code or inherit player input and UI they don't want.

**After:** Extract Class into focused components.

```csharp
public class Health : MonoBehaviour
{
    public int maxHealth = 100;
    public int Current { get; private set; }
    public event Action<int> Changed;

    void Awake() => Current = maxHealth;

    public void TakeDamage(int amount)
    {
        Current = Mathf.Max(Current - amount, 0);
        Changed?.Invoke(Current);
        if (Current == 0) Destroy(gameObject);
    }
}

public class HealthUI : MonoBehaviour
{
    [SerializeField] Health health;
    [SerializeField] Text label;

    void OnEnable() => health.Changed += Show;
    void OnDisable() => health.Changed -= Show;
    void Show(int hp) => label.text = $"HP: {hp}";
}
```

Movement and inventory get the same treatment (`PlayerMovement`, `Inventory`). Now **any** GameObject, including enemies or breakable crates, can use `Health`. The UI updates through an event, so `Health` no longer needs to know the UI exists.

### Python: A Tournament Manager

**Before:** one class handles players, team balancing, results, and output.

```python
class Tournament:
    def __init__(self, name):
        self.name = name
        self.players = []
        self.wins = {}

    def add_player(self, name, rank):
        self.players.append({"name": name, "rank": rank})

    def make_teams(self, count):
        ranked = sorted(self.players, key=lambda p: p["rank"], reverse=True)
        return [ranked[i::count] for i in range(count)]

    def record_win(self, team):
        self.wins[team] = self.wins.get(team, 0) + 1

    def announce(self):
        print(f"🏆 {self.name} Standings")
        for team, wins in sorted(self.wins.items(), key=lambda w: w[1], reverse=True):
            print(f"Team {team}: {wins} wins")
```

Each new feature, like scheduling, Discord posts, or saving to a file, would land in this same class. And because `announce()` prints directly, the standings can't be tested or posted anywhere else.

**✅ After:** Extract Class for each responsibility.

```python
class Roster:
    def __init__(self):
        self.players = []

    def add(self, name, rank):
        self.players.append({"name": name, "rank": rank})

    def balanced_teams(self, count):
        ranked = sorted(self.players, key=lambda p: p["rank"], reverse=True)
        return [ranked[i::count] for i in range(count)]


class Standings:
    def __init__(self):
        self.wins = {}

    def record_win(self, team):
        self.wins[team] = self.wins.get(team, 0) + 1

    def ranked(self):
        return sorted(self.wins.items(), key=lambda w: w[1], reverse=True)


def format_standings(title, standings):
    lines = [f"🏆 {title} Standings"]
    lines += [f"Team {team}: {wins} wins" for team, wins in standings.ranked()]
    return "\n".join(lines)


class Tournament:
    def __init__(self, name):
        self.name = name
        self.roster = Roster()
        self.standings = Standings()

    def announce(self):
        print(format_standings(self.name, self.standings))
```

`Tournament` is now a small class that coordinates output from others, but the printed output is unchanged. `format_standings` returns a string, so the same text can be printed, posted to Discord, or checked in a test. Team balancing can also be tested without recording any results.


## References

- Martin Fowler, [*Refactoring*](https://martinfowler.com/books/refactoring.html), 2nd ed., Ch. 2–3
- [Refactoring.Guru: Large Class](https://refactoring.guru/smells/large-class)
- [Refactoring.Guru: Extract Class](https://refactoring.guru/extract-class)
