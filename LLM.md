# Code-Driven LLM Architecture

This document explains the core philosophy and implementation patterns for building reliable AI agents where **code controls the LLM**, not the other way around.

## The Fundamental Principle

Traditional chatbots rely on prompt engineering to control behavior. The LLM is given instructions and trusted to follow them. This approach is inherently unreliable because LLMs:

- Forget instructions mid-conversation
- Misinterpret ambiguous rules
- Hallucinate capabilities or data
- Behave inconsistently across sessions

**The code-driven approach inverts this relationship.** Instead of asking the LLM to remember rules, we embed all business logic in SWAIG functions. The LLM becomes a natural language interface that translates user intent into function calls, while the functions handle all the actual work.

```
┌─────────────────────────────────────────────────────────────┐
│                    Traditional Approach                     │
│                                                             │
│   User ──► LLM (with rules in prompt) ──► Response          │
│                                                             │
│   Problem: LLM must remember and apply rules correctly      │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   Code-Driven Approach                      │
│                                                             │
│   User ──► LLM ──► SWAIG Function ──► Response to LLM       │
│                         │                                   │
│                         ▼                                   │
│                   Code enforces:                            │
│                   • Business rules                          │
│                   • State management                        │
│                   • Calculations                            │
│                   • Validation                              │
│                   • UI updates                              │
│                                                             │
│   Advantage: LLM only translates intent, code does the rest │
└─────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. SWAIG Functions

SWAIG (SignalWire AI Gateway) functions are the bridge between natural language and deterministic code. When the LLM determines user intent, it calls a function. The function executes business logic and returns a response that guides the LLM's reply.

```python
@self.tool(
    name="add_item",
    description="Add an item to the order",
    parameters={
        "type": "object",
        "properties": {
            "item_name": {"type": "string", "description": "Name of the menu item"},
            "quantity": {"type": "integer", "description": "How many to add", "minimum": 1, "maximum": 10}
        },
        "required": ["item_name"]
    }
)
def add_item(args, raw_data):
    # All business logic lives here, not in the prompt
    order_state, global_data = get_order_state(raw_data)

    # Code validates limits (LLM doesn't need to know about limits)
    if quantity > MAX_ITEMS_PER_TYPE:
        quantity = MAX_ITEMS_PER_TYPE

    # Code finds items (LLM doesn't need to know the menu structure)
    sku, item_data, category = find_menu_item(item_name)

    # Code calculates prices (LLM never does math)
    order_state["subtotal"], order_state["tax"], order_state["total"] = calculate_totals(order_state["items"])

    # Code detects upsell opportunities (LLM doesn't need complex rules)
    combo_suggestion = check_combo_opportunity(order_state["items"])

    # Return response that guides LLM behavior
    response = f"Added {item_data['name']} to your order."
    if combo_suggestion:
        response += f"\n\n{combo_suggestion}"  # LLM will naturally offer this

    return SwaigFunctionResult(response)
```

**Key insight:** The function's response text shapes what the LLM says next. By including a combo suggestion in the response, we guide the LLM to offer it without needing prompt rules about when to upsell.

### 2. SwaigFunctionResult

The `SwaigFunctionResult` object is how functions communicate back to the system. It's not just a text response—it can:

- Update global state
- Trigger UI events
- Change conversation state
- Control the agent's next actions

```python
result = SwaigFunctionResult("Added Beef Taco to your order.")

# Persist state for future function calls
result.update_global_data(global_data)

# Send real-time updates to the frontend
result.swml_user_event({
    "type": "item_added",
    "item": {"name": "Beef Taco", "price": 3.49},
    "order_total": 3.84
})

# Transition the state machine
result.swml_change_step("taking_order")

return result
```

### 3. State Machine (Contexts and Steps)

The state machine controls what the LLM can do at any point in the conversation. Each state defines:

- **Available functions**: What actions the LLM can take
- **Valid transitions**: Where the conversation can go next
- **Context-specific prompts**: Additional instructions for the current state

```python
contexts = self.define_contexts()

default_context = contexts.add_context("default") \
    .add_section("Goal", "Take accurate food orders efficiently")

# Greeting state - limited actions
default_context.add_step("greeting") \
    .add_section("Current Task", "Welcome the customer") \
    .set_functions(["add_item"]) \           # Can only add items
    .set_valid_steps(["taking_order"])       # Can only go to taking_order

# Taking order state - more actions available
default_context.add_step("taking_order") \
    .add_section("Current Task", "Build the customer's order") \
    .add_bullets("Rules", [
        "Current total: $${global_data.order_state.total}",  # Dynamic data injection
        "If customer says 'done': CALL finalize_order"
    ]) \
    .set_functions(["add_item", "remove_item", "finalize_order", "cancel_order"]) \
    .set_valid_steps(["confirming_order"])
```

**Why this matters:** By restricting available functions per state, we prevent impossible actions. The LLM literally cannot call `process_payment` until we're in the confirming state.

### 4. Global Data

Global data persists across function calls within a conversation. It's how functions share state without the LLM needing to track anything.

```python
# Initialize at agent startup
self.set_global_data({
    "order_state": {
        "items": [],
        "total": 0.00,
        "subtotal": 0.00,
        "tax": 0.00,
        "order_number": None,
        "item_count": 0
    }
})

# Read in any function
def add_item(args, raw_data):
    global_data = raw_data.get('global_data', {})
    order_state = global_data.get('order_state', default_order)

# Update and persist
def save_order_state(result, order_state, global_data):
    global_data['order_state'] = order_state
    result.update_global_data(global_data)
```

**Dynamic prompt injection:** Global data can be referenced in step definitions using `${global_data.path.to.value}` syntax. This lets the prompt reflect current state without the LLM needing to ask.

```python
.add_bullets("Rules", [
    "Current order has ${global_data.order_state.item_count} items",
    "Current total: $${global_data.order_state.total}"
])
```

## Design Patterns

### Pattern 1: Response-Guided Behavior

Instead of telling the LLM when to do something, make functions return responses that naturally lead the LLM to do it.

**Bad approach (prompt-based):**
```
If the customer orders 2 tacos, chips, and a drink, and they haven't
already ordered a combo, calculate the savings and offer to upgrade
them to a Taco Combo which costs $9.99 instead of $11.96...
```

**Good approach (response-guided):**
```python
def add_item(args, raw_data):
    # ... add the item ...

    # Check for combo opportunity
    combo_suggestion = check_combo_opportunity(order_state["items"])

    response = f"Added {item_data['name']}."
    if combo_suggestion:
        # This text guides the LLM to offer the upgrade
        response += f"\n\nGreat news! I can upgrade to a Taco Combo and save you $1.97!"

    return SwaigFunctionResult(response)
```

The LLM receives this response and naturally communicates it to the user. No complex prompt rules needed.

### Pattern 2: Code-Enforced Limits

Never trust the LLM to enforce limits. Validate everything in code.

```python
def add_item(args, raw_data):
    quantity = args.get("quantity", 1)

    # Hard limits enforced in code
    MAX_ITEMS_PER_TYPE = 20
    MAX_TOTAL_ITEMS = 50
    MAX_ORDER_VALUE = 500.00

    # Silently cap excessive quantities
    if quantity > 10:
        quantity = 10
        limited_message = " (Limited to 10 per add)"

    # Check total items
    current_total = sum(item["quantity"] for item in order_state["items"])
    if current_total + quantity > MAX_TOTAL_ITEMS:
        quantity = MAX_TOTAL_ITEMS - current_total
        limited_message = f" (Limited to stay within {MAX_TOTAL_ITEMS} items)"

    # Check order value
    potential_total = order_state["subtotal"] + (item_data["price"] * quantity)
    if potential_total > MAX_ORDER_VALUE:
        return SwaigFunctionResult(f"This would exceed our ${MAX_ORDER_VALUE} limit.")
```

### Pattern 3: Fuzzy Input Handling

Users speak naturally. Code must interpret their intent flexibly.

```python
# Menu aliases handle variations
MENU_ALIASES = {
    "D003": ["water", "bottled water", "water bottle", "aqua", "h2o"],
    "S001": ["chips", "chips and salsa", "salsa and chips", "nachos"],
}

def find_menu_item(item_name):
    item_lower = item_name.lower().strip()

    # 1. Exact match
    for category, items in MENU.items():
        for sku, item_data in items.items():
            if item_lower == item_data["name"].lower():
                return sku, item_data, category

    # 2. Alias match
    for sku, aliases in MENU_ALIASES.items():
        if item_lower in [a.lower() for a in aliases]:
            # Look up the actual item
            ...

    # 3. TF-IDF similarity (handles typos, partial matches)
    if HAS_SKLEARN:
        user_vector = self.vectorizer.transform([item_lower])
        similarities = cosine_similarity(user_vector, self.menu_vectors)[0]
        if max(similarities) > 0.42:
            return best_match

    # 4. Fuzzy word matching (fallback)
    ...
```

### Pattern 4: State Transitions in Code

Functions control state transitions, not the LLM.

```python
def add_item(args, raw_data):
    # ... process the item ...

    result = SwaigFunctionResult(response)

    # Code decides to transition from greeting to taking_order
    result.swml_change_step("taking_order")

    return result

def finalize_order(args, raw_data):
    # ... validate order ...

    result = SwaigFunctionResult("Does everything look correct?")

    # Code transitions to confirmation
    result.swml_change_step("confirming_order")

    return result
```

### Pattern 5: UI Synchronization

Functions push updates to the UI in real-time, keeping the visual state in sync without LLM involvement.

```python
def add_item(args, raw_data):
    # ... add item to order ...

    result = SwaigFunctionResult(response)

    # Push update to frontend
    result.swml_user_event({
        "type": "item_added",
        "item": {
            "sku": sku,
            "name": item_data["name"],
            "quantity": quantity,
            "price": item_data["price"],
            "total": round(item_data["price"] * quantity, 2)
        },
        "order_total": order_state["total"],
        "subtotal": order_state["subtotal"],
        "tax": order_state["tax"],
        "item_count": order_state["item_count"]
    })

    return result
```

The frontend receives these events via WebRTC and updates the display. The LLM doesn't need to describe the order—the user sees it directly.

### Pattern 6: Descriptive Function Parameters

Help the LLM call functions correctly with clear parameter descriptions.

```python
@self.tool(
    name="remove_item",
    description="Remove an item from the order",
    parameters={
        "type": "object",
        "properties": {
            "item_name": {
                "type": "string",
                "description": "Name of item to remove"
            },
            "quantity": {
                "type": "integer",
                "description": "How many to remove (default: 1, use -1 for all)",
                "minimum": -1
            }
        },
        "required": ["item_name"]
    }
)
```

### Pattern 7: Instruction Embedding in Steps

When the LLM needs behavioral guidance, embed it in step definitions rather than the main prompt.

```python
default_context.add_step("taking_order") \
    .add_bullets("IMPORTANT RULES", [
        "When customer orders multiple items: CALL add_item FOR EACH ITEM SEPARATELY",
        "CRITICAL: If customer says 'X and Y', you MUST call add_item twice",
        "If add_item response includes 'Great news!' about a combo:",
        "  - If customer says 'yes': CALL upgrade_to_combo",
        "NEVER quote prices yourself - let the functions provide them"
    ])
```

These rules are injected into the prompt only when in the `taking_order` state, keeping the prompt focused and relevant.

## The Mindset

### 1. The LLM is a Translator

Think of the LLM as a sophisticated intent recognizer and natural language generator. Its job is to:
- Understand what the user wants
- Call the appropriate function
- Communicate the function's response naturally

It should NOT:
- Remember business rules
- Perform calculations
- Track state
- Make decisions about limits or validation

### 2. Functions Are the Source of Truth

Everything that matters happens in functions:
- Current order state
- Valid menu items
- Price calculations
- Business rule enforcement
- UI updates

If you need reliable behavior, put it in a function.

### 3. Responses Shape Conversation

The text returned by functions directly influences what the LLM says. Use this to:
- Guide the LLM toward specific actions ("Would you like to upgrade to a combo?")
- Provide accurate information (prices, totals, item names)
- Handle errors gracefully ("I couldn't find that item. Please check the menu.")

### 4. State Machines Prevent Chaos

By defining explicit states with allowed functions and transitions:
- Impossible actions become literally impossible
- The conversation follows predictable paths
- Recovery from errors is straightforward

### 5. Prompts Are Personality, Not Logic

Use prompts for:
- Tone and personality ("You're warm and enthusiastic")
- Communication style ("Keep responses concise")
- High-level role ("You're a drive-thru order taker")

Don't use prompts for:
- Business rules (use function logic)
- Calculations (use function code)
- State management (use global_data)
- Decision trees (use state machine)

## Summary

The code-driven approach produces AI agents that are:

- **Reliable**: Business logic is deterministic, not probabilistic
- **Testable**: Functions can be unit tested
- **Maintainable**: Changes to rules don't require prompt engineering
- **Auditable**: Execution path is clear and logged
- **Consistent**: Same input produces same output

The LLM handles what it's good at (natural language), while code handles what it's good at (logic, state, validation). This division of labor produces agents that users can actually depend on.
