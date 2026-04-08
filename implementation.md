# Code Generation and Modification Guidelines

**Purpose**: Reference for applying spec changes with insertion rules, code generation patterns, and immediate verification.

**When to use**: When implementing ADD, MODIFY, and REMOVE operations on Quint specs.

---

## Insertion Point Decision Matrix

### TYPE Definitions

| Scenario | Insertion Point | Detection Method |
|----------|----------------|------------------|
| **After imports** | Last line starting with `import` + 1 blank line | `grep -n "^import " <spec_path> \| tail -1` |
| **Before state vars** | Line before first `var` declaration - 1 | `grep -n "^var " <spec_path> \| head -1` |
| **Before definitions** | Line before first `def` or `pure def` - 1 | `grep -n "^\\(pure \\)\\?def " <spec_path> \| head -1` |
| **Fallback** (no vars/defs) | After module opening brace + 1 blank line | `grep -n "module.*{" <spec_path>` + 2 |

**Decision Logic**:
```
Find: last_import_line = grep "^import " | tail -1
Find: first_var_line = grep "^var " | head -1
Find: first_def_line = grep "^def " | head -1

If last_import_line exists:
  insertion_point = last_import_line + 2
Else if first_var_line exists:
  insertion_point = first_var_line - 1
Else if first_def_line exists:
  insertion_point = first_def_line - 1
Else:
  Find: module_line = grep "module.*{"
  insertion_point = module_line + 2
```

### STATE VARS

| Scenario | Insertion Point | Detection Method |
|----------|----------------|------------------|
| **After types** | Last type definition + 1 blank line | `grep -n "^type " <spec_path> \| tail -1` |
| **Before pure defs** | Line before first `pure def` - 1 | `grep -n "^pure def " <spec_path> \| head -1` |
| **Group with vars** | After last existing `var` declaration | `grep -n "^var " <spec_path> \| tail -1` |
| **Fallback** | After types section or before definitions | Use TYPE fallback logic |

**Decision Logic**:
```
Find: last_var_line = grep "^var " | tail -1
Find: last_type_line = grep "^type " | tail -1
Find: first_pure_def_line = grep "^pure def " | head -1

If last_var_line exists (group with existing vars):
  insertion_point = last_var_line + 1
Else if last_type_line exists:
  insertion_point = last_type_line + 2
Else if first_pure_def_line exists:
  insertion_point = first_pure_def_line - 1
Else:
  Use TYPE fallback
```

### PURE DEFS

| Scenario | Insertion Point | Detection Method |
|----------|----------------|------------------|
| **After state vars** | Last `var` declaration + 1 blank line | `grep -n "^var " <spec_path> \| tail -1` |
| **Before actions** | Line before first non-pure `def` - 1 | `grep -n "^def [^=]*=" <spec_path> \| head -1` |
| **Group related** | Near similar pure defs (same domain) | Read context, place near related functions |

**Decision Logic**:
```
Find: last_var_line = grep "^var " | tail -1
Find: last_pure_def_line = grep "^pure def " | tail -1
Find: first_action_line = grep "^def " (not "pure def") | head -1

If last_pure_def_line exists AND related:
  insertion_point = last_pure_def_line + 1
Else if last_var_line exists:
  insertion_point = last_var_line + 2
Else if first_action_line exists:
  insertion_point = first_action_line - 1
Else:
  After state vars section
```

### ACTIONS

| Scenario | Insertion Point | Detection Method |
|----------|----------------|------------------|
| **After pure defs** | Last `pure def` + 1 blank line | `grep -n "^pure def " <spec_path> \| tail -1` |
| **Near related action** | Adjacent to action being modified | If modifying `step`, place near `step` |
| **Before init** | Line before `def init` - 1 | `grep -n "^def init" <spec_path>` |
| **Fallback** | After all pure defs, before tests | End of definitions section |

**Decision Logic**:
```
If modifying existing action:
  Find: target_action_line = grep "def <action_name>"
  insertion_point = target_action_line + <action_length> + 1
Else:
  Find: last_pure_def_line = grep "^pure def " | tail -1
  Find: init_line = grep "^def init"

  If last_pure_def_line exists:
    insertion_point = last_pure_def_line + 2
  Else if init_line exists:
    insertion_point = init_line - 1
  Else:
    After state vars section
```

---

## Code Generation Protocol

### Step 1: Query KB for Syntax (if unfamiliar)

**When**: Generating new code element with uncertain syntax

**Commands**:
```bash
# For specific constructs
quint_get_example("union type definition")
quint_get_example("state variable with map type")
quint_get_doc("action definition syntax")

# For patterns
quint_hybrid_search("quint <element_type> syntax examples")
```

**Decision**:
```
If element_type in [unfamiliar_types]:
  Query KB: quint_get_example("<element_type>")
  Use returned syntax exactly
Else:
  Use known syntax pattern
```

### Step 2: Generate Code with Proper Formatting

**Indentation Detection**:
```
Read file at insertion area (5 lines before, 5 lines after)
Count leading spaces/tabs in existing definitions
Use same indentation style (spaces vs tabs, width)
```

**Blank Line Rules**:
```
Before new definition: Always add 1 blank line
After new definition:
  If followed by another definition: Add 1 blank line
  If last in section: No blank line
```

**Example Generation** (Union Type):

**Input** (from refactor plan):
```json
{
  "item": "type",
  "name": "TimeoutEvent",
  "details": "Union type: RequestTimeout | PrepareTimeout | CommitTimeout"
}
```

**Generation Logic**:
```
1. Parse details: Extract variant names
2. Query KB if needed: quint_get_example("union type")
3. Generate structure:
   type <name> =
     | <variant1>
     | <variant2>
     | <variant3>
4. Apply indentation (detect from file)
```

**Output**:
```quint
type TimeoutEvent =
  | RequestTimeout
  | PrepareTimeout
  | CommitTimeout
```

### Step 3: Insert with Edit Tool

**Insertion Strategy**:
```
1. Read file at insertion_point (get context)
2. Identify anchor text (line to insert before/after)
3. Construct old_string (anchor text)
4. Construct new_string (new code + anchor text)
5. Execute: Edit <file>, old_string, new_string
```

**Example**:
```
Insertion point: Line 30 (before first var)
Anchor text: "var round: Round"

old_string = "var round: Round"
new_string = "type TimeoutEvent =\n  | RequestTimeout\n  | PrepareTimeout\n  | CommitTimeout\n\nvar round: Round"

Edit consensus.qnt, old_string, new_string
```

---

## Modification Patterns

### Pattern 1: Adding to `any{}` Choices

**Detection**:
```
Plan details contains: "Add <action> to <target> choices"
Target action has: any { ... }
```

**Strategy**:
```
1. Find target action: grep "def <target>"
2. Read action definition: Read <file> lines <start>:<end>
3. Locate closing brace of any{}
4. Insert before closing brace with comma
```

**Original**:
```quint
def step = any {
  propose,
  prevote
}
```

**Modification**:
```
old_string = "  prevote\n}"
new_string = "  prevote,\n  precommit\n}"
```

**Result**:
```quint
def step = any {
  propose,
  prevote,
  precommit
}
```

### Pattern 2: Adding to `all{}` Conditions

**Original**:
```quint
def propose = all {
  condition1,
  condition2
}
```

**Modification**:
```
old_string = "  condition2\n}"
new_string = "  condition2,\n  not(hasTimeout(round))\n}"
```

### Pattern 3: Extending State Variable Type

**Original**:
```quint
var state: { round: Int, votes: Set[Vote] }
```

**Modification**:
```
old_string = "var state: { round: Int, votes: Set[Vote] }"
new_string = "var state: { round: Int, votes: Set[Vote], timeouts: Set[TimeoutEvent] }"
```

### Pattern 4: Wrapping Existing Action

**Original**:
```quint
def propose =
  updateState(...)
```

**Modification**:
```
old_string = entire action
new_string = "def propose = all {\n  not(hasTimeout(round)),\n  updateState(...)\n}"
```

---

## Immediate Verification Protocol

### After EVERY Change

**Typecheck**
```
Execute: quint typecheck <spec_path>
Check: exit_code == 0

If exit_code != 0:
  See iteration.md for Type Error Recovery
Else:
  Record: Change verified successfully
```

---

## Verification Checklist

Execute after each change (mandatory):

```
Step 1: Typecheck verification
  [ ] quint typecheck <spec_path> returns exit_code 0

  If fails: See iteration.md

Step 2: Change documentation
  [ ] Record: Changed <item_name> at line <X>
  [ ] Record: Verification status (parse ok, typecheck ok)
```

---

## Optional Verification Steps

**When**: After successful parse + typecheck, for high-risk changes

### LSP Hover Verification
```
Purpose: Verify types are as expected

Execute: mcp__quint-lsp__textDocument/hover {position: at new definition}
Check: Returned type matches expected type from plan
```

### Reference Verification (for MODIFY operations)
```
Purpose: Ensure all references still valid

Execute: mcp__quint-lsp__textDocument/references {position: at modified element}
Check: All references still valid (no broken references)
```

### Quick Simulation
```
Purpose: Ensure spec is executable

Execute: quint run <spec_path> --max-steps=10 --max-samples=10 --backend=rust
Check: No runtime errors in first 5 steps
```

---

## Escalation Criteria

Stop and ask user for guidance if:

| Condition | Trigger | Action |
|-----------|---------|--------|
| Max Retry Attempts Exceeded | parse_error_attempts >= 5 | Revert last change, report, ask for guidance |
| Semantic Ambiguity | Refactor plan details unclear | Ask for clarification |
| Breaking Change Detected | Typecheck fails in unrelated code | Report what broke, ask to continue |
| Unsafe Removal | Removing element with references | Report references, ask to continue |

---

## Tool Usage Decision Matrix

| Task | Tool | Command Pattern |
|------|------|-----------------|
| Find insertion point | Grep | `grep -n "^<keyword> " <file>` |
| Read context | Read | `Read <file> lines <start>:<end>` |
| Find references | LSP | `mcp__quint-lsp__textDocument/references` |
| Check type | LSP | `mcp__quint-lsp__textDocument/hover` |
| Get syntax example | KB | `quint_get_example("<construct>")` |
| Apply change | Edit | `Edit <file>, old_string, new_string` |
| Verify parse | Bash | `quint parse <file>` |
| Verify types | Bash | `quint typecheck <file>` |
| Quick test | Bash | `quint run <file> --max-steps=5` |
