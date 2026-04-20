# Tricentis Tosca TSU File Format — Complete Reference

## 1. Container Format

A `.tsu` file is a **gzip-compressed JSON document** that represents a full or partial export of a Tosca test workspace.

### Decompression

```
file.tsu.gzip  →  gunzip  →  file.tsu  (UTF-8 JSON, no line breaks, single text node)
```

The decompressed file is a single-line ASCII/UTF-8 text file. The top-level JSON object has exactly one key:

```json
{ "Entities": [ ...N objects... ] }
```

All content — projects, folders, test cases, modules, parameters, files — lives in this flat array. There is no nesting between entities; relationships are expressed exclusively through UUID cross-references.

---

## 2. Entity Envelope

Every object in the `Entities` array follows this schema:

```json
{
  "ObjectClass": "<string>",
  "Surrogate":   "<uuid>",
  "Attributes":  { "<key>": "<string-value>", ... },
  "Assocs":      { "<key>": ["<surrogate>", ...], ... }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `ObjectClass` | string | The entity type name (see §3) |
| `Surrogate` | string | Globally unique identifier. Two formats are used: standard UUID (`3a1cb69a-b792-fc34-e633-50065d6e121e`) and ULID (`01KGJ6D109AND66ZQBCSEX7GKF`). Both serve as opaque cross-reference keys. |
| `Attributes` | object | All scalar properties. **Every value is a string**, even numbers and booleans. |
| `Assocs` | object | Named reference lists. Each key maps to an array of Surrogate strings pointing to other entities. An empty array means the relationship is absent. |

### Common Attribute Fields (appear on most entity types)

| Attribute | Meaning |
|-----------|---------|
| `Name` | Human-readable label |
| `Description` | Free-text description, may be empty |
| `CheckoutWorkspace` | Name of the workspace that has the entity checked out; empty if not checked out |
| `CheckOutState` | `"1"` = checked in (available), other values = locked |
| `SynchronizationPolicy` | `"2"` = synchronize with server (always `"2"` in practice) |
| `Revision` | Integer revision counter as string; increments on each save |
| `IncludeForSynchronization` | `"1"` = included in sync operations |

---

## 3. Entity Types

### 3.1 Project Structure

#### `TCProject`
The single root node of the workspace.

**Attributes:** `Name`, `CheckoutWorkspace`, `SynchronizationPolicy`, `Description`, `CheckOutState`, `Revision`, `IncludeForSynchronization`, `SpecialProjectName`

**Assocs:**

| Key | Points to |
|-----|-----------|
| `Items` | `TCComponentFolder` — top-level component containers |
| `ConfigurationLinks` | `TCConfigurationLink` |
| `Properties`, `AttachedFiles` | internal metadata |
| `Groups`, `Users`, `UserBookmarks`, `ObjectPropertiesDefinitions` | workspace-level user/permission objects (typically empty in exports) |

---

#### `TCComponentFolder`
A top-level partition of the project tree (e.g. "TestCases", "Modules", "Configurations").

**Attributes:** `Name`, `CheckoutWorkspace`, `SynchronizationPolicy`, `Description`, `CheckOutState`, `Revision`, `IncludeForSynchronization`, `ContentPolicy`

`ContentPolicy` is a compact string encoding which entity types are allowed as children (Tosca-internal format).

**Assocs:**

| Key | Points to |
|-----|-----------|
| `Items` | `TCFolder`, `TCConfiguration`, `TestStepLibrary`, etc. |
| `Project` | `TCProject` (parent) |
| `ParentFolder` | `TCComponentFolder` (if nested) |
| `ExecutionEntryFolders`, `ObjectPropertiesDefinitions` | execution/metadata (typically empty) |

---

#### `TCFolder`
A recursive folder node that organises test cases and sub-folders.

**Attributes:** `Name`, `CheckoutWorkspace`, `SynchronizationPolicy`, `Description`, `CheckOutState`, `Revision`, `IncludeForSynchronization`, `ContentPolicy`, `TestConfigurationParameters`

`TestConfigurationParameters` — see §4.1 for decoding.

**Assocs:**

| Key | Points to |
|-----|-----------|
| `ParentFolder` | `TCFolder` or `TCComponentFolder` |
| `Items` | `TestCase`, `TCFolder`, `OwnedRecoveryScenarioCollection` |
| `ConfigurationLinks` | `TCConfigurationLink` |
| `Project` | `TCProject` (only on direct children of the project root) |

---

#### `TCConfiguration`
A named configuration profile applied to test execution.

**Attributes:** `Name`, `CheckoutWorkspace`, `SynchronizationPolicy`, `Description`, `CheckOutState`, `Revision`, `IncludeForSynchronization`, `Predefined`, `ProvidesNamespace`, `TestConfigurationParameters`

`Predefined`: `"0"` = user-defined, `"1"` = built-in Tosca configuration.  
`ProvidesNamespace`: `"0"` = no, `"1"` = this configuration contributes a namespace to TDS lookups.  
`TestConfigurationParameters` — see §4.1.

**Assocs:**

| Key | Points to |
|-----|-----------|
| `ParentFolder` | `TCComponentFolder` |
| `UsedBy` | `TCConfigurationLink` — every folder/item that references this config |
| `ParentConfiguration` | `TCConfiguration` — for inherited configurations |
| `Items` | child `TCConfiguration` entries (sub-configurations) |

---

#### `TCConfigurationLink`
Junction record linking a configured item to a configuration.

**Attributes:** `Name` (always empty)

**Assocs:**

| Key | Points to |
|-----|-----------|
| `ConfiguredItem` | `TCFolder` or other configurable entity |
| `UsedConfiguration` | `TCConfiguration` |

---

### 3.2 Test Cases

#### `TestCase`
A single automated test scenario.

**Attributes:** `Name`, `CheckoutWorkspace`, `SynchronizationPolicy`, `Description`, `CheckOutState`, `Revision`, `IncludeForSynchronization`, `Pausable`, `TestCaseWorkState`, `IsBusinessTestCase`, `DerivedFromName`

| Attribute | Values |
|-----------|--------|
| `Pausable` | `"1"` = can be paused during execution |
| `TestCaseWorkState` | `"0"` = not started / draft, `"1"` = in progress, `"2"` = complete/ready |
| `IsBusinessTestCase` | `"0"` = technical test case, `"1"` = business-level test case |
| `DerivedFromName` | name of the template this was derived from; empty if original |

**Assocs:**

| Key | Points to |
|-----|-----------|
| `ParentFolder` | `TCFolder` |
| `Items` | `TestStepFolder` — the execution phases (e.g. Precondition, Test, Cleanup) |
| `ExecutionEntries`, `ExecutionLogs` | execution history (empty in this export type) |
| `ReferencedBy` | other `TestCase` entities that reference this one |

---

#### `TestStepFolder`
A named phase within a test case. Conventionally: Precondition, Test, Cleanup.

**Attributes:** `Name`, `Condition`, `Path`, `BreakInstantiation`, `Repetition`, `DisabledDescription`, `Pausable`

The `Condition`, `Path`, `Repetition` fields are runtime control fields (see §3.4 for their meaning on step containers). `Pausable`: `"0"` = not pausable, `"2"` = pausable.

**Assocs:**

| Key | Points to |
|-----|-----------|
| `TestCase` | `TestCase` (owning test case) |
| `Items` | `TestStepFolderReference`, `TestCaseControlFlowItem` |
| `ParentFolder` | `TestStepFolder` (for nested phases) |

---

### 3.3 Reusable Step Library

#### `TestStepLibrary`
A named container for a collection of reusable step blocks. The library is the top-level organisational unit within the module/library tree.

**Attributes:** `Name`, `CheckoutWorkspace`, `SynchronizationPolicy`, `Description`, `CheckOutState`, `Revision`, `IncludeForSynchronization`

**Assocs:**

| Key | Points to |
|-----|-----------|
| `ReusableItems` | `ReuseableTestStepBlock` |
| `ParentFolder` | `TCFolder` or `TCComponentFolder` |

---

#### `ReuseableTestStepBlock`
A named, reusable sequence of test steps — the Tosca equivalent of a callable function or page-object method.

**Attributes:** `Name`, `Description`, `Condition`, `Path`, `BreakInstantiation`, `Repetition`, `DisabledDescription`, `Pausable`

**Assocs:**

| Key | Points to |
|-----|-----------|
| `Library` | `TestStepLibrary` (owning library) |
| `Items` | `XTestStep`, `TestCaseControlFlowItem` — the steps inside this block |
| `UsedBy` | `TestStepFolderReference` — every call-site that invokes this block |
| `ParameterLayer` | `ParameterLayer` — the parameter contract for this block |

---

#### `TestStepFolderReference`
A call-site: an occurrence of a `ReuseableTestStepBlock` inside a `TestStepFolder`. This is the most numerous entity type because the same block is reused across many test cases.

**Attributes:** `Name`, `Condition`, `Path`, `BreakInstantiation`, `Repetition`, `DisabledDescription`, `Pausable`

`Name` mirrors the block's name for readability. `Pausable` is always `"0"` on references.

**Assocs:**

| Key | Points to |
|-----|-----------|
| `ParentFolder` | `TestStepFolder` or `TestCaseControlFlowFolder` |
| `ReusedItem` | `ReuseableTestStepBlock` — the block being called |
| `ParameterLayerReference` | `ParameterLayerReference` — the bound parameter values for this call |
| `TestCase` | `TestCase` (back-reference, sometimes populated) |

---

### 3.4 Execution Steps

The following attributes appear on all execution containers (`XTestStep`, `TestCaseControlFlowItem`, `TestCaseControlFlowFolder`, `TestStepFolderReference`, `ReuseableTestStepBlock`, `TestStepFolder`):

| Attribute | Meaning |
|-----------|---------|
| `Condition` | A boolean expression; if false, the step/block is skipped. Empty = always run. |
| `Path` | XPath-like path to a sub-element; used by some engine-level steps. Usually empty. |
| `BreakInstantiation` | `"0"` = continue on failure, `"1"` = abort the test case on failure |
| `Repetition` | A repeat count or loop expression; empty = execute once |
| `DisabledDescription` | If non-empty, the step is disabled and this text explains why |
| `Pausable` | `"0"` = not pausable, `"2"` = execution engine may pause here for manual inspection |

---

#### `XTestStep`
A single executable test step — the atomic unit of execution. Each step targets one `XModule` (or `ApiModule`) and carries one or more `XTestStepValue` parameter assignments.

**Attributes:** `Name`, `Condition`, `Path`, `BreakInstantiation`, `Repetition`, `DisabledDescription`, `Pausable`, `ReorderAllowed`

`ReorderAllowed`: `"0"` = fixed position, `"1"` = Tosca may reorder during optimisation.

**Assocs:**

| Key | Points to |
|-----|-----------|
| `ParentFolder` | `ReuseableTestStepBlock`, `RecoveryScenario`, or `TestCaseControlFlowFolder` |
| `Module` | `XModule` or `ApiModule` — the UI/API target |
| `TestStepValues` | `XTestStepValue` — one entry per module attribute being acted on |
| `TestCase` | `TestCase` (back-reference when step lives in a test case directly) |

---

#### `XTestStepValue`
One parameter assignment within a step: maps a value to a specific module attribute and defines what action to perform.

**Attributes:** `ExplicitName`, `DataType`, `ActionMode`, `Operator`, `Value`, `ActionProperty`, `DisabledDescription`, `Condition`, `Path`

`ExplicitName`: overrides the module attribute's name in the report; usually empty.  
`ActionProperty`: auxiliary property string used by some engine actions; usually empty.

##### `ActionMode` (enum)

Derived from observed data. Tosca encodes the action type as an integer bitmask:

| Value | Inferred meaning |
|-------|-----------------|
| `1` | **Input / Read** — read a value from the UI into a buffer |
| `37` | **Input / Set** — the default "perform action" mode; types text, sets options, triggers waits |
| `69` | **Click** — interacts with a control (button, checkbox, toggle); `Value` is `True`/`False` |
| `101` | **WaitOn / Verify Exists** — waits until the element is present/visible; `Value` is `True`/`False` |
| `165` | **Verify** — asserts the element's value matches |
| `515` | **TDS Input** — reads from Test Data Service into the step |
| `517` | **TDS Output** — writes from the step back to Test Data Service |
| `519` | **TDS Write** — writes a value directly to a TDS attribute |

##### `DataType` (enum)

| Value | Meaning |
|-------|---------|
| `0` | String (default) |
| `2` | Numeric / Integer |
| `3` | Date/Time |
| `4` | Boolean |

##### `Operator` (enum)

| Value | Meaning |
|-------|---------|
| `0` | Equals (default) |
| `1` | Contains |
| `2` | Not equals / Greater than |

##### `Value` — Expression Language

Plain string literals are used as-is. The Tosca expression language uses `{...}` syntax:

| Expression | Meaning |
|-----------|---------|
| `{NULL}` | Null / empty / no value |
| `{B[<name>]}` | Read from the named **buffer** variable |
| `{TDS[<sheet>.<column>]}` | Read from **Test Data Service** at the specified sheet/column |
| `{PL[<name>]}` | Read from a **parameter layer** variable named `<name>` |
| `{CLICK}` / `{C[...]}` | Emit a click action |
| `{TEXTINPUT[<expr>]}` | Type text resolved from `<expr>` |
| `{SENDKEYS["<keys>"]}` | Send raw keyboard input (supports `^{a}` notation) |
| `{ENTER}` | Press Enter key |
| `{BACKSPACE}` | Press Backspace key |
| `{F5}` etc. | Function key press |
| `{LEFT}` | Left arrow key |
| `{CLEAR}` | Clear the field |
| `{MOUSEOVER}` | Hover the mouse over the element |
| `{TRIM[<expr>]}` | Trim whitespace from `<expr>` |
| `{MATH[<expr>]}` | Evaluate arithmetic expression (e.g. `{MATH[{B[x]}+10]}`) |
| `{RND[<n>]}` | Random numeric string of length `<n>` |
| `{DATE[][][<format>]}` | Current date formatted per `<format>` (e.g. `dd/MM/yyyy`) |
| `{REGEX["<pattern>"]}` | Match against a regular expression |
| `{NUMBEROFOCCURRENCES[<val>][<sub>]}` | Count occurrences of `<sub>` within `<val>` |
| `{LEFT[...]}` | Left substring operation |
| `{STRINGREPLACE[<s>][<find>][<replace>]}` | String replacement |
| `{XB[<name>]}<suffix>` | Extended buffer read with optional string suffix |

**Assocs:**

| Key | Points to |
|-----|-----------|
| `TestStep` | `XTestStep` (owning step) |
| `ModuleAttribute` | `XModuleAttribute` — the specific field/control being acted on |
| `SubValues` | `XTestStepValue` — nested values for hierarchical attributes |
| `ParentValue` | `XTestStepValue` — parent when this is a sub-value |

---

#### `TestCaseControlFlowItem`
A control-flow statement (If or While loop) embedded in a test step sequence.

**Attributes:** `Name`, `Condition`, `Path`, `BreakInstantiation`, `Repetition`, `DisabledDescription`, `Pausable`, `StatementType`, `MaximumRepetitions`

##### `StatementType` (ControlFlowItem)

| Value | Meaning |
|-------|---------|
| `1` | **If** statement |
| `3` | **While** loop |

`MaximumRepetitions`: maximum loop iterations for While; empty for If.

**Assocs:**

| Key | Points to |
|-----|-----------|
| `ParentFolder` | `ReuseableTestStepBlock`, `TestStepFolder`, or `TestCaseControlFlowFolder` |
| `ControlFlowFolders` | `TestCaseControlFlowFolder` — ordered list: [Condition, Then] or [Condition, Then, Else] |

---

#### `TestCaseControlFlowFolder`
One branch of a control-flow item: the condition expression, the true branch, or the false branch.

**Attributes:** `Name`, `Condition`, `Path`, `BreakInstantiation`, `Repetition`, `DisabledDescription`, `Pausable`, `StatementType`

##### `StatementType` (ControlFlowFolder)

| Value | Meaning |
|-------|---------|
| `0` | **Condition** — the boolean expression to evaluate |
| `1` | **Then** — steps executed when condition is true |
| `2` | **Else** — steps executed when condition is false (optional) |

**Assocs:**

| Key | Points to |
|-----|-----------|
| `ParentControlFlowItem` | `TestCaseControlFlowItem` |
| `Items` | `XTestStep`, `TestStepFolderReference`, `TestCaseControlFlowItem` |

---

### 3.5 Technical Modules

#### `XModule`
Represents a UI screen, component, or element group. Modules are the "map" of the application under test — they describe what can be interacted with and how.

**Attributes:** `Name`, `Description`, `TCProperties`, `CheckoutWorkspace`, `SynchronizationPolicy`, `CheckOutState`, `Revision`, `IncludeForSynchronization`, `InterfaceType`, `ImplementationType`, `TechnicalId`, `IsAbstract`, `ValueSelectionGroup`, `BusinessType`, `SpecialIcon`

| Attribute | Values / Meaning |
|-----------|-----------------|
| `InterfaceType` | `"0"` = framework/utility module (TBox, PDF, etc.), `"1"` = web UI module |
| `ImplementationType` | Engine-specific implementation class; usually empty |
| `TechnicalId` | Engine-assigned identifier; usually empty |
| `IsAbstract` | `"0"` = concrete, `"1"` = abstract base module |
| `ValueSelectionGroup` | Groups modules for value-selection UI in Tosca Commander; usually empty |
| `BusinessType` | Business classification label; usually empty |
| `SpecialIcon` | Custom icon identifier; usually empty |
| `TCProperties` | **Base64 gzip XML** — see §4.2 |

**Assocs:**

| Key | Points to |
|-----|-----------|
| `ParentFolder` | `TCFolder` or `TCComponentFolder` |
| `Attributes` | `XModuleAttribute` — the fields/controls on this module |
| `TestSteps` | `XTestStep` — every step that uses this module |
| `Specializations` | `XModule` — derived module variants |
| `UsedAsDefaultSpecializationIn` | `XModule` — modules where this is the default variant |
| `ReferencingAttributes` | `XModuleAttribute` — attributes in other modules that reference this one |

---

#### `XModuleAttribute`
A single interactive element on a module: a button, text field, dropdown, checkbox, etc.

**Attributes:** `Name`, `Description`, `ValueSelectionGroup`, `BusinessType`, `SpecialIcon`, `Cardinality`, `DefaultDataType`, `DefaultActionMode`, `InterfaceType`, `DefaultValue`, `Visible`, `TCProperties`

| Attribute | Values / Meaning |
|-----------|-----------------|
| `Cardinality` | How many instances may exist: `"0-1"` (optional, singular), `"1"` (required, singular), `"0-N"` (optional, multiple), `"1-N"` (required, multiple) |
| `DefaultDataType` | Default `DataType` for step values targeting this attribute (same enum as §3.4) |
| `DefaultActionMode` | Default `ActionMode` pre-filled when a step is created (same enum as §3.4) |
| `InterfaceType` | `"0"` = framework attribute, `"1"` = web element, `"2147483647"` = special/universal attribute |
| `DefaultValue` | The value pre-filled in new step values; usually empty |
| `Visible` | `"1"` = shown in the UI, `"0"` = hidden |
| `TCProperties` | **Base64 gzip XML** — see §4.2 (rarely set; carries `ScanTag` when present) |

**Assocs:**

| Key | Points to |
|-----|-----------|
| `Module` | `XModule` (owning module) |
| `TestStepValues` | `XTestStepValue` — every step value that targets this attribute |
| `Attributes` | `XModuleAttribute` — child attributes (hierarchical attribute trees) |
| `UIChildren` | `XModuleAttribute` — UI-level children |
| `ParentAttribute` | `XModuleAttribute` — parent in the attribute tree |
| `UIParent` | `XModuleAttribute` — UI-level parent |
| `AdditionalInfo`, `Attachments` | metadata attachments |

---

#### `ApiModule`
A module representing a single HTTP API call. Always comes in request/response pairs, linked by `CorrelationId` in their `TCProperties` blob.

**Attributes:** `Name`, `Description`, `TCProperties`, `CheckoutWorkspace`, `SynchronizationPolicy`, `CheckOutState`, `Revision`, `IncludeForSynchronization`, `InterfaceType`, `ImplementationType`, `TechnicalId`, `IsAbstract`, `ValueSelectionGroup`, `BusinessType`, `SpecialIcon`, `ExplicitConnection`, `Headers`, `Payload`, `Script`

The four API-specific fields are all **base64-encoded** (plain, not gzip):

| Attribute | Decoded format | Content |
|-----------|---------------|---------|
| `ExplicitConnection` | JSON array of `{Key, Value}` pairs | Connection metadata: `Name` (`"<explicit>"` for explicit URLs) and `Endpoint` (the full URL) |
| `Headers` | JSON array of arrays of `{Key, Value}` pairs | HTTP headers; each inner array is one header as `[{Key:"Key", Value:"<name>"}, {Key:"Value", Value:"<val>"}]` |
| `Payload` | UTF-8 string | The raw HTTP request body (typically JSON) |
| `Script` | UTF-8 string | Optional script for request pre-processing; usually empty |
| `TCProperties` | **Base64 gzip XML** — see §4.3 |

The `Headers` field on a **response** `ApiModule` (where `TCProperties.IsRequest=False`) contains the captured response headers verbatim.

**Assocs:** same as `XModule`.

---

### 3.6 Parameters and Data Binding

#### `ParameterLayer`
A named set of parameter definitions attached to a `ReuseableTestStepBlock` — analogous to a function's parameter list.

**Attributes:** `Name`

**Assocs:**

| Key | Points to |
|-----|-----------|
| `ReuseableTestStepBlock` | `ReuseableTestStepBlock` (owning block) |
| `Parameters` | `Parameter` — the individual parameter definitions |
| `ParameterLayerReferences` | `ParameterLayerReference` — all call-site bindings |

---

#### `Parameter`
One parameter definition within a `ParameterLayer`.

**Attributes:** `Name`, `ValueSelectionGroup`, `Description`

**Assocs:**

| Key | Points to |
|-----|-----------|
| `ParameterLayer` | `ParameterLayer` (owning layer) |
| `ParameterReferences` | `ParameterReference` — all values bound to this parameter across call-sites |
| `Parameters` | `Parameter` — child parameters (hierarchical parameter trees) |

---

#### `ParameterLayerReference`
The binding of a `ParameterLayer` at a specific call-site (`TestStepFolderReference`). One per layer per call-site.

**Attributes:** `Name` (mirrors the layer name)

**Assocs:**

| Key | Points to |
|-----|-----------|
| `TestStepFolderReference` | `TestStepFolderReference` (the call-site) |
| `ParameterLayer` | `ParameterLayer` (the layer being bound) |
| `AllParameterReferences` | `ParameterReference` — all value bindings for this call |

---

#### `ParameterReference`
One actual value bound to one `Parameter` at a specific call-site.

**Attributes:** `Value` — a literal string or a Tosca expression (e.g. `{TDS[sheet.column]}`, `{B[name]}`, `{PL[name]}`)

**Assocs:**

| Key | Points to |
|-----|-----------|
| `ParameterLayerReference` | `ParameterLayerReference` (owning binding) |
| `Parameter` | `Parameter` (the parameter being set) |

---

#### `XParam`
An extended property attached to an `XModule` or `XModuleAttribute`. Carries metadata that the core `Attributes` object does not have a dedicated field for.

**Attributes:** `Name`, `Value`, `Visible`, `Readonly`, `SupressCopy`, `ParamType`

`Visible`: `"1"` / `"0"`. `Readonly`: `"1"` / `"0"`. `SupressCopy`: `"1"` = exclude from copy operations.

##### `ParamType` (enum)

| Value | Meaning | Example `Name` = `Value` |
|-------|---------|--------------------------|
| `2` | **SelfHealingData** — serialised JSON blob (`Tricentis.TCCore…TcSelfHealingData`); see §4.4 | `SelfHealingData = {...}` |
| `4` | **Display text / localisation label** — a human-readable caption used for self-healing identification | `Text = "button label"`, `Label = "field name"` |
| `5` | **UI locator property** — a single named property used in self-healing (AutomationId, ClassName, ControlTypeName, etc.) | `Name = "Submit"`, `ClassName = "MdButton"` |
| `7` | **XPath / complex selector** — an XPath expression used as a fallback locator | `Xpath = "//div[@id='x']"` |
| `8` | **Engine / framework property** — configuration flags for the Tosca engine | `Parameter = "true"`, `Engine = "Framework"`, `ScanTag = "TBoxWait"` |

**Assocs:**

| Key | Points to |
|-----|-----------|
| `ExtendableObject` | `XModule` or `XModuleAttribute` — the entity this property decorates |

---

### 3.7 Attached Files

#### `OwnedFile`
Metadata record for a file attached to a test artifact.

**Attributes:** `Name`, `FileName`, `FilePath`, `FileExtension`, `FileType`, `CheckoutWorkspace`, `SynchronizationPolicy`, `Description`, `CheckOutState`, `Revision`, `IncludeForSynchronization`

| Attribute | Meaning |
|-----------|---------|
| `FileName` | Filename without extension |
| `FilePath` | Original path on disk at capture time; usually empty in exports |
| `FileExtension` | File extension including dot (e.g. `".png"`) |
| `FileType` | `"0"` = embedded file (always `"0"` in observed data) |

**Assocs:**

| Key | Points to |
|-----|-----------|
| `AttachedTo` | The entity this file is attached to (e.g. `TestCase`) |
| `EmbeddedContent` | `FileContent` — the corresponding content record |

---

#### `FileContent`
The binary content of one attached file, stored as base64.

**Attributes:** `Data`, `ExternalFileId`, `ExternalFileDirectory`, `ExternalFileRevision`

`Data`: base64-encoded raw binary. Decode with standard base64 to recover the original bytes. Common file types (by magic bytes): `89504E47` = PNG, `25504446` = PDF.  
`ExternalFileId`, `ExternalFileDirectory`, `ExternalFileRevision`: used when the file is stored externally rather than inline; empty when `Data` is populated.

**Assocs:**

| Key | Points to |
|-----|-----------|
| `File` | `OwnedFile` — the metadata record |

---

### 3.8 Recovery Scenarios

#### `OwnedRecoveryScenarioCollection`
Container for all recovery scenarios in a folder subtree. There is at most one per folder.

**Attributes:** standard version/checkout fields only  
**Assocs:** `Scenarios` → `RecoveryScenario` list; `ParentFolder` → `TCFolder`

---

#### `RecoveryScenario`
A named error-recovery procedure executed automatically when a test step fails.

**Attributes:** `Name`, `Condition`, `Path`, `BreakInstantiation`, `Repetition`, `DisabledDescription`, `Pausable`, `RetryLevel`, `ScenarioType`

| Attribute | Values |
|-----------|--------|
| `RetryLevel` | `"2"` = retry at test-step level |
| `ScenarioType` | `"0"` = standard recovery |

**Assocs:**

| Key | Points to |
|-----|-----------|
| `Items` | `XTestStep`, `TestCaseControlFlowItem` — recovery steps |
| `OwnedScenarioCollection` | `OwnedRecoveryScenarioCollection` |

---

## 4. Encoded / Embedded Sub-formats

### 4.1 `TestConfigurationParameters` — Base64 Gzip XML

**Appears on:** `TCFolder`, `TCConfiguration`  
**Encoding:** base64 → gunzip → UTF-8 XML

XML root element depends on entity type:
- `TCFolder` → `<TestConfigurationParameterCollection>`
- `TCConfiguration` → `<TestConfigurationParameterCollection>`

Each child is a `<TCProperty>` element:

```xml
<TCProperty
  Name="Browser"
  Value="Chrome"
  Visible="False"
  SupressCopy="False"
  ValueType="String"
  ValueSelectionGroup=""
  ExportToSubset="True"
/>
```

**Attribute fields on `<TCProperty>`:**

| Attribute | Meaning |
|-----------|---------|
| `Name` | The configuration parameter name |
| `Value` | The parameter value (may itself be base64 — see below) |
| `Visible` | `"True"` / `"False"` — whether shown in the UI |
| `SupressCopy` | `"True"` = excluded from copy/clone operations |
| `ValueType` | `"String"` (always in observed data) |
| `ValueSelectionGroup` | Picker group for dropdown selection; usually empty |
| `ExportToSubset` | `"True"` = included when exporting a subset |

**Known `TCProperty` names seen in `TCFolder`:**

| Name | Example value | Purpose |
|------|--------------|---------|
| `Browser` | `Chrome` | Execution browser |
| `SynchronizationTimeout` | `30000` | Global sync timeout in ms |
| `SynchronizationTimeoutDuringWaitOn` | `80000` | WaitOn-specific sync timeout in ms |

**Known `TCProperty` names seen in `TCConfiguration`:**

| Name | Value / Purpose |
|------|----------------|
| `Browser` | Target browser name |
| `SynchronizationTimeout` | Global sync timeout in ms |
| `AuthenticationAccessKey` | **Nested base64 JSON** (see §4.1.1) — Tosca TDS OAuth2 credentials |
| `TestDataEndpoint` | URL of the Test Data Service |
| `TestDataRepository` | Name of the TDS repository to connect to |
| `OnDialogFailure` | Failure handling: `"Recover"` = invoke recovery scenario |
| `OnExceptionFailure` | Exception handling: `"Recover"` |
| `OnVerificationFailure` | Verification failure handling: `"Recover"` |
| `TestCaseRetries` | Number of full test-case retry attempts |
| `TestStepRetries` | Number of step-level retry attempts |
| `TestStepValueRetries` | Number of value-level retry attempts |

#### 4.1.1 Nested Base64 in `AuthenticationAccessKey`

The `Value` attribute of the `AuthenticationAccessKey` `<TCProperty>` is itself **base64-encoded plain UTF-8 JSON**:

```json
{
  "ClientId": "<oauth2-client-id>",
  "DisplayName": "<display-name>",
  "ClientSecret": "<oauth2-client-secret>",
  "Scopes": [
    "AuthenticationServiceApi",
    "DexApi",
    "FileServiceApi",
    "ProjectServiceApi",
    "TestDataServiceApi",
    "ToscaAutomationObjectServiceApi",
    "AdminConsoleApi",
    "MigrationServiceApi",
    "ExecutionResultServiceApi",
    "RpaApiGatewayApi",
    "TestDesignStudioApi",
    "NexusApi",
    "OsvStudioApi",
    "LiveCompareServiceApi",
    "DataIntegrityServiceApi",
    "NotificationServiceApi",
    "ExampleServiceApi",
    "SapIntegrationApi",
    "MbtServiceApi",
    "HubServiceApi",
    "GatewayApi",
    "LicenseAdministrationApi",
    "visionai.nexus.api",
    "visionai.agent.api",
    "authentication.groups.read"
  ]
}
```

> **Security note:** This blob contains an OAuth2 client secret in plaintext. Anyone with access to the TSU file can extract these credentials without any additional key.

---

### 4.2 `TCProperties` on `XModule` / `XModuleAttribute` — Base64 Gzip XML

**Encoding:** base64 → gunzip → UTF-8 XML  
**XML root:** `<TCPropertiesCollection>`

Uses the same `<TCProperty>` element schema as §4.1 but with `Visible="True"` and `Readonly` field.

```xml
<TCPropertiesCollection>
  <TCProperty
    Name="Version"
    Value="10.1"
    Visible="True"
    Readonly="False"
    SupressCopy="False"
    ValueType="String"
    ValueSelectionGroup=""
    ExportToSubset="True"
  />
</TCPropertiesCollection>
```

**Known property names:**

| Name | Appears on | Meaning |
|------|-----------|---------|
| `Version` | `XModule`, `ApiModule` | Tosca module engine version (e.g. `"10.1"`, `"13.3"`, `"24.2"`) |
| `ScanTag` | `XModule`, `XModuleAttribute` | Engine scan identifier used during module scanning (e.g. `"TBoxWait"`, `"TBoxSendKeysKeys"`) |

---

### 4.3 `TCProperties` on `ApiModule` — Base64 Gzip XML

Uses the same encoding as §4.2 but carries API-specific properties. `ApiModule` entities always come in **request/response pairs** — both share the same `CorrelationId`.

**Request module** (`IsRequest = "True"`):

| Name | Meaning |
|------|---------|
| `IsRequest` | `"True"` |
| `CorrelationId` | UUID linking this request to its paired response module |
| `Version` | Tosca HTTP engine version |
| `Method` | HTTP method (`POST`, `GET`, etc.) |
| `InactiveNodes` | How to handle disabled payload nodes: `"Remove"` |
| `ListSupport` | List handling strategy: `"Dynamic"` |

**Response module** (`IsRequest = "False"`):

| Name | Meaning |
|------|---------|
| `IsRequest` | `"False"` |
| `CorrelationId` | Same UUID as the paired request module |
| `Version` | Tosca HTTP engine version |
| `ResponseTime` | Recorded response time in ms (from the capture session) |
| `StatusCode` | HTTP status code and reason (e.g. `"200 OK"`) |
| `Direction` | `"In"` for responses |
| `InactiveNodes` | `"Remove"` |

---

### 4.4 `XParam` — SelfHealingData JSON (`ParamType = 2`)

The `Value` field is a **plain JSON string** (not base64, not compressed) serialised using .NET JSON.NET conventions with `$type` discriminators.

```json
{
  "$type": "Tricentis.TCCore.BusinessObjects.Modules.SelfHealing.TcSelfHealingData, BusinessObjects",
  "HealingParameters": {
    "$type": "System.Collections.Generic.List`1[[...TcSelfHealingProperty...]]",
    "$values": [
      {
        "$type": "...TcSelfHealingProperty, BusinessObjects",
        "Name": "<property-name>",
        "Surrogate": "<uuid>",
        "ParamType": 4,
        "Value": "<captured-value>",
        "Weight": 1.0
      }
    ]
  }
}
```

Each `$values` entry is a self-healing property captured during module scanning:

| `Name` | `ParamType` | Meaning |
|--------|-------------|---------|
| `Caption` | 4 | Visible text caption of the element |
| `Text` | 4 | Inner text content |
| `Name` | 5 | `name` attribute |
| `AutomationId` | 5 | `id` / automation ID attribute |
| `ClassName` | 5 | CSS class name |
| `ControlTypeName` | 5 | Tosca control type (e.g. `ControlType.Button`) |
| `FrameworkId` | 5 | Browser/framework name (e.g. `Chrome`) |
| `Orientation` | 5 | `OrientationType_None` or directional value |
| `Xpath` | 7 | Fallback XPath expression |

`Weight` is a float between 0 and 1 representing the confidence weight for this property in the self-healing algorithm.

---

### 4.5 `ApiModule` Inline Fields — Plain Base64

`ExplicitConnection`, `Headers`, and `Payload` are base64-encoded plain UTF-8 (not gzip). Decode with standard base64.

**`ExplicitConnection`** decodes to a JSON array:
```json
[
  {"Key": "Name",     "Value": "<explicit>"},
  {"Key": "Endpoint", "Value": "https://..."}
]
```
`Name` is the literal string `"<explicit>"` when the URL is specified directly rather than via a named connection.

**`Headers`** decodes to a JSON array of arrays, one per header:
```json
[
  [{"Key": "Key", "Value": "Content-Type"}, {"Key": "Value", "Value": "application/json"}],
  [{"Key": "Key", "Value": "Accept"},       {"Key": "Value", "Value": "*/*"}]
]
```
On response modules this contains the full set of HTTP response headers captured during the API scan session. These headers may include session cookies and security tokens.

**`Payload`** decodes to a raw UTF-8 string — typically a JSON body for POST requests, or the full HTTP response body for response modules.

> **Security note:** Response `Headers` arrays often contain live `Set-Cookie` and session token values captured during the API scan. `Payload` may contain sensitive response data. Both are stored in cleartext.

---

## 5. Entity Relationship Graph

```
TCProject
  └─ Items ──► TCComponentFolder (top-level partitions)
                 └─ Items ──► TCFolder (recursive)
                                └─ Items ──► TestCase
                                │              └─ Items ──► TestStepFolder (Precondition/Test/Cleanup)
                                │                             └─ Items ──► TestStepFolderReference
                                │                             │              └─ ReusedItem ──► ReuseableTestStepBlock
                                │                             │              └─ ParameterLayerReference
                                │                             │                    └─ AllParameterReferences ──► ParameterReference
                                │                             │                                                        └─ Parameter ──► ParameterLayer
                                │                             └─ Items ──► TestCaseControlFlowItem
                                │                                              └─ ControlFlowFolders ──► TestCaseControlFlowFolder
                                │                                                                            └─ Items ──► (same as above)
                                └─ Items ──► TestStepLibrary
                                               └─ ReusableItems ──► ReuseableTestStepBlock
                                                                        └─ Items ──► XTestStep
                                                                        │              └─ Module ──► XModule / ApiModule
                                                                        │              └─ TestStepValues ──► XTestStepValue
                                                                        │                                        └─ ModuleAttribute ──► XModuleAttribute
                                                                        └─ ParameterLayer ──► Parameter

XModule / XModuleAttribute
  └─ Properties ──► XParam (metadata decorators)

TestCase / XModule / ApiModule / OwnedFile
  └─ AttachedFiles ──► OwnedFile
                           └─ EmbeddedContent ──► FileContent

TCFolder / TCConfiguration
  └─ ConfigurationLinks ──► TCConfigurationLink
                                └─ UsedConfiguration ──► TCConfiguration

TCFolder ──► OwnedRecoveryScenarioCollection
                └─ Scenarios ──► RecoveryScenario
                                     └─ Items ──► XTestStep (recovery steps)
```

---

## 6. Surrogate ID Formats

Two ID formats co-exist in the same file:

| Format | Example | Notes |
|--------|---------|-------|
| UUID v4 (hyphenated) | `3a1cb69a-b792-fc34-e633-50065d6e121e` | Used by entities created in older Tosca versions |
| ULID | `01KGJ6D109AND66ZQBCSEX7GKF` | Used by entities created in newer Tosca versions; lexicographically sortable by creation time |

Both formats are treated identically as opaque strings. Cross-references always use the Surrogate value exactly as stored — no normalisation is performed.

---

## 7. Known Gaps and Limitations

| Area | Gap |
|------|-----|
| `ActionMode` bitmask | The full Tosca SDK enum definition is not embedded in the file. Values documented in §3.4 are inferred from data patterns. |
| `CheckOutState` | Values other than `"1"` (checked-out states) are not present in typical read-only exports. |
| `ContentPolicy` | The compact encoding on `TCFolder` / `TCComponentFolder` is Tosca-internal and not documented in the file. |
| `ExecutionLogs` / `ExecutionEntries` | These assoc lists are always empty in exported TSU files; execution history is stored separately in Tosca's execution results service. |
| `FileContent.Data` empty records | ~90 of 463 file records have an empty `Data` field; this indicates the file existed but content was excluded from the export (possibly due to size limits or export settings). |
| `FileContent.ExternalFileId` | Infrastructure for external file storage is present in the schema but unused in observed data. |
| `XParam.ParamType` values `4` and `5` | The distinction between ParamType 4 (display text) and 5 (locator property) maps to the `Weight` applied in self-healing: `4` properties receive weight `1.0`, `5` properties receive `0.5`. |
| `TCProject.Groups` / `Users` | User and group membership is exported as empty arrays; access control data is managed server-side. |
