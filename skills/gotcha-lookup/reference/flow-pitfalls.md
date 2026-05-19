# Salesforce Flow & Apex Pitfalls

> Curated catalog of common Salesforce automation gotchas. Each entry follows the format: **What happened → Root cause → Prevention**.
>
> Stable IDs: once an entry is numbered, the number is permanent even if the entry is archived. New entries take the next available number.

---

### #1 — Flow "Fast Update" fails when one field is FLS-restricted

- **What happened**: A Screen Flow threw `INVALID_FIELD_FOR_INSERT_UPDATE` when updating a custom object via a "Fast Update" element, even though all field values were valid.
- **Root cause**: "Fast Update" saves the *entire* `sObjectVariable` back to the database. If the running user lacks FLS "Edit" access on **even one** field in that payload (often dozens of fields wide), the entire DML fails. Salesforce does not silently skip restricted fields.
- **Prevention**: On objects with non-trivial FLS, avoid "Fast Update" (whole-record `<inputReference>`) and use targeted `<inputAssignments>` to set only the fields that actually need to change. This caps your FLS surface area to the fields you're explicitly updating.

---

### #2 — Visualforce-embedded flow doesn't receive the record ID

- **What happened**: A Screen Flow launched from a `<flow:interview>` component on a Visualforce page had no record context — `recordId` was null.
- **Root cause**: The `<flow:interview>` tag was missing the `<apex:param>` block that maps a VF page expression into the Flow's input variable. Flow doesn't auto-bind context from the surrounding page.
- **Prevention**: Always include `<apex:param name="recordId" value="{!YourObject__c.Id}"/>` inside `<flow:interview>` when the flow expects a record ID. The variable name must match the Flow's input variable exactly.

---

### #3 — Screen Flow "Unhandled Fault" with no useful detail

- **What happened**: A Screen Flow failed in the UI with a generic "Unhandled Fault" toast and no error path to inspect. `FlowExecutionErrorEvent` is not queryable.
- **Root cause**: Screen Flows do not expose Apex-style stack traces in the UI by default. The actual error lives in the Apex Debug Log, behind a TraceFlag.
- **Prevention**: Set a User TraceFlag for the test user (`USER_DEBUG` + `WORKFLOW` categories), reproduce the failure, then read the Debug Log — it contains the failing element name and exception. Don't waste time hunting through Flow Builder.

---

### #4 — Flow "duplicate check" on a record collection always evaluates true

- **What happened**: A Decision element checking "duplicate exists" using `{!CollectionVar.IsNull} = False` always returned true, even when the prior Get Records returned zero records.
- **Root cause**: In Flow, querying for multiple records produces a **collection variable**. An empty collection is *not* null — `IsNull` on a collection returns `false` whether it has 0 or 1000 records.
- **Prevention**: Either count the collection (`{!CollectionVar.Count} > 0`) or change the Get Records element to store **only the first record** (single sObject variable). Single sObject variables *are* null when nothing is returned.

---

### #5 — Quick Action flow can't access the current record

- **What happened**: A Screen Flow launched from a Quick Action couldn't read the record it was launched from — the input variable was always null.
- **Root cause**: Quick Action passes the launching record's ID into a Flow variable named exactly `recordId`. The flow had named the variable `varCustomerFeedbackId` instead.
- **Prevention**: When building a Flow for use with a Quick Action, declare a text input variable named exactly **`recordId`** with "Available for input" checked. The name is a Salesforce convention, not configurable on the Quick Action side.

---

### #6 — Auto-number / formula field is blank right after Screen Flow Create

- **What happened**: A Screen Flow created a record then displayed the auto-number Name on the next screen. The displayed value was blank.
- **Root cause**: Auto-Number and Formula fields are computed **on commit**. Screen Flows pause the transaction between elements, so the auto-number isn't materialized yet when the next screen renders.
- **Prevention**: After the Create element, add a Get Records element keyed on the newly-created record's `Id` to fetch the materialized auto-number/formula values, then bind the screen to those refreshed values.

---

### #7 — `Get Records` for a Record Type fails with INVALID_CROSS_REFERENCE_KEY

- **What happened**: A Flow filtering by `RecordType.DeveloperName = 'General_Liability_Claim'` threw `INVALID_CROSS_REFERENCE_KEY` in a target environment, despite passing in source.
- **Root cause**: The Record Type's DeveloperName differs between orgs — what's `General_Liability_Claim` in one org may be just `GL` in another. Org admins rename Record Types and the DeveloperName follows.
- **Prevention**: Before hardcoding a DeveloperName in any Flow filter, run `sf data query --query "SELECT DeveloperName FROM RecordType WHERE SobjectType='YourObject__c'" --use-tooling-api --target-org <org>` and confirm. Better: look up the Record Type Id at runtime via Get Records.

---

### #8 — Assumed field API name doesn't match reality

- **What happened**: A Flow mapping wrote to `Claimant_Address__c`, but the actual API name in the org was `Claimant_Address_Line1__c`. Deploy succeeded; runtime mapping silently produced null.
- **Root cause**: Field API names diverge from field labels, especially after renames or when fields were created across multiple sprints by different admins. Flow's "field picker" can mask this if you typed the API name manually.
- **Prevention**: Verify exact API names via `FieldDefinition` query before hardcoding into Flow expressions, formula fields, or Apex: `SELECT QualifiedApiName FROM FieldDefinition WHERE EntityDefinition.QualifiedApiName='YourObject__c'`.

---

### #9 — Formula field source returns null because the formula references a missing field

- **What happened**: A Flow mapping from a formula field produced null on the target record. The Flow logic was correct.
- **Root cause**: The source formula referenced fields that don't exist (or are inaccessible to the running user) — the formula silently evaluated to null instead of erroring.
- **Prevention**: When a Flow mapping unexpectedly produces null, don't immediately blame the Flow. Open the source field on the source record in the UI — if the value is null there too, the bug is in the formula definition, not the Flow.

---

### #10 — Testing flows with partially-populated records causes false passes

- **What happened**: A Flow's logic passed manual tests on a hand-crafted test record but failed in production on real records that had additional fields populated.
- **Root cause**: Hand-crafted test records typically populate only the fields the developer remembers. Real records have dozens of additional fields populated, some of which the Flow reads (often via formulas or rollups) and behaves differently with.
- **Prevention**: Build "robust" test records via CLI (`sf data create record`) populating **every** field the Flow touches — including fields read by formula fields the Flow consumes. Cross-reference against the Flow's "Variables and Resources" list, not memory.

---

### #11 — Flow activation fails with cryptic `null__NotFound`

- **What happened**: Deploying a Flow succeeded but activation failed with a generic `null__NotFound` error and no field/element pointed to.
- **Root cause**: Multiple possible causes, all of which produce the same opaque error:
  1. The Flow XML references a field that doesn't exist in the target org.
  2. The Flow assigns or filters on a Record Type Id that exists but has `<active>false</active>`.
  3. The user activating the Flow lacks FLS Read on a field used in a lookup filter or assignment.
- **Prevention**: When you hit `null__NotFound`, check in this order: (a) every field referenced in the Flow XML exists in the target org, (b) every Record Type used is `<active>true</active>`, (c) the activating user has FLS on every field touched. A temporary "Flow Builder All Access" Permission Set often unblocks (c).

---

### #12 — `Get Records` crashes when its filter variable is null

- **What happened**: A `Get Records` element using a variable in its filter threw a runtime exception when the variable was null (empty lookup field on the source record).
- **Root cause**: Some Flow filter operations cannot accept a null right-hand side. The Flow does not auto-skip — it errors.
- **Prevention**: Wrap any `Get Records` whose filter depends on an optional lookup in a Decision element first: "Is lookup populated? → Yes → run Get / No → skip". Make null-safety explicit, not implicit.

---

### #13 — Inserting numeric data into an SSN field fails because the field is type Double

- **What happened**: An Apex script inserting test data assigned string values to `SSN__c` (e.g., `"123-45-6789"`) and got a compile/type error.
- **Root cause**: Some "obviously a string" fields (SSN, phone, account number) are configured as `Number(9,0)` or `Double` in the schema, not Text. The display masks this.
- **Prevention**: Before writing Apex test data or DataWeave mappings for sensitive fields, run `sf sobject describe --sobject YourObject__c --target-org <org>` and confirm the `type` of every field you're populating. Don't trust the UI's display format.

---

### #14 — `<apexPluginCalls>` in Flow XML needs a `Process.Plugin` class, not `@InvocableMethod`

- **What happened**: A Flow containing `<apexPluginCalls>` failed to deploy because the referenced Apex class was missing in the target org. The team deployed an `@InvocableMethod` class with the same name and re-deployed — still failed.
- **Root cause**: `<apexPluginCalls>` is a **legacy** Flow mechanism that invokes Apex classes implementing the `Process.Plugin` interface. It is *not* the same as `<actionCalls>` which invokes `@InvocableMethod`. The two are different Apex extension models with different class shapes.
- **Prevention**: When you see `<apexPluginCalls>` in Flow XML, find the `<apexClass>` child element, then verify the class exists and implements `Process.Plugin`. Migration consideration: rewrite as `<actionCalls>` + `@InvocableMethod` only if you're prepared to rewrite the Apex.

---

### #15 — Managed vs. unmanaged namespace prefix on migrated metadata (`pkg__` vs `pkg_`)

- **What happened**: A Flow retrieved from an org with a managed package referenced `pkg__SomeObject__c` and `pkg__SomeField__c`. After deploy to a target org where the package was rebuilt as unmanaged, the Flow failed: those API names are now `pkg_SomeObject__c` and `pkg_SomeField__c`.
- **Root cause**: Managed packages use a double-underscore prefix (`namespace__ApiName__c`). Unmanaged rebuilds of the same metadata typically drop the double underscore to a single one (`namespace_ApiName__c`) since the platform reserves `__` for managed package namespacing.
- **Prevention**: When migrating Flow/Apex XML between orgs with different package install states, grep for the managed prefix (`pkg__`) and bulk-replace with the unmanaged equivalent (`pkg_`). Validate after replace: managed and unmanaged objects with similar names can both exist and you don't want to clobber the wrong one.

---

### #16 — First Flow activation fails generically because dependencies weren't deployed first

- **What happened**: First-time activation of a Flow on a newly-deployed object failed with a generic error that pointed to no specific field or element.
- **Root cause**: When multiple dependencies (fields + Record Types + Permission Set FLS) are all missing simultaneously, Salesforce returns a single opaque error rather than enumerating them.
- **Prevention**: Deploy the full dependency stack in order **before** the first activation attempt:
  1. Custom Fields
  2. Record Types (with `<active>true</active>`)
  3. Permission Sets granting FLS to the deploying/activating user
  4. THEN the Flow

  Activating a Flow in a half-deployed state will succeed silently in some cases and fail opaquely in others. Don't gamble — finish the stack.

---

### #17 — A field-length error often hides spam or a broken upstream integration

- **What happened**: A Web-to-Case flow creating Customer Feedback records started failing with `STRING_TOO_LONG: Zip Code: data value too large: +12159137691 (max length=10)`. The obvious fix was to widen the Zip Code field from 10 to 20 chars.
- **Root cause**: The submitting Case wasn't a real customer — it was a spam bot. Multiple unrelated fields (Address, City, First Name, Last Name, Zip, Batch) all contained the same phone number `+12159137691`, and the Description was an Ahrefs affiliate ad with an `is.gd` URL shortener. The Zip field happened to be the shortest, so it errored first — but every other field had absorbed garbage silently. Widening Zip would have let the spam record through into the CR queue, making things worse for downstream agents.
- **Prevention**: When a Flow fails with a field-level error (STRING_TOO_LONG, INVALID_TYPE, REQUIRED_FIELD_MISSING), **read every field value in the error email, not just the one that triggered the error**. Red flags for spam-bot Web-to-Case: identical values across unrelated fields, HTML anchor tags or URL shorteners in the Description, dot-stuffed Gmails (`b.e.t.t.i.e@gmail.com`), phone numbers in name/address fields. Real fix: add a spam-filter Decision element upstream of the Create Records (skip create if Description contains `<a `/`href=`/`is.gd`/`bit.ly`, or if First Name == Last Name == Zip). Don't widen fields to silence errors that are doing useful filtering for you.

---

## Contributing a new gotcha

If you've hit a Salesforce automation gotcha not catalogued here, please open an issue at https://github.com/shantanudutta1/salesforce-flow-sense/issues with:

1. **What happened** — the concrete symptom (error text, runtime behavior)
2. **Root cause** — the non-obvious mechanism, ideally with a reference to Salesforce docs or a Stack Exchange thread
3. **Prevention** — a checkable step, not generic advice

Entries that don't separate symptom from cause from prevention will be asked to be reformatted before merge — the structure is what makes the catalog useful.
