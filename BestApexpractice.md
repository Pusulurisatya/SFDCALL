# Salesforce Apex Development — Best Practices Prompt

> **Usage:** Add this file to your project root as `.github/copilot-instructions.md` for GitHub Copilot, or attach it when prompting Claude/ChatGPT to generate Apex classes.

---

## 🔒 CORE RULES (Non-Negotiable)

When generating any Apex class, trigger, or service, you MUST follow all rules below without exception.

---

## 1. BULKIFICATION — Always Handle 200+ Records

Every method must process **collections**, never single records. Triggers fire with up to 200 records per execution.

```apex
// ❌ NEVER DO THIS
for (Account acc : accounts) {
    List<Contact> contacts = [SELECT Id FROM Contact WHERE AccountId = :acc.Id]; // SOQL in loop
    update contacts; // DML in loop
}

// ✅ ALWAYS DO THIS
Set<Id> accountIds = new Set<Id>();
for (Account acc : accounts) { accountIds.add(acc.Id); }

List<Contact> contacts = [SELECT Id, AccountId FROM Contact WHERE AccountId IN :accountIds];
for (Contact c : contacts) { c.Description = 'Updated'; }
update contacts; // Single DML
```

**Rule:** Never put SOQL queries or DML statements inside a `for` loop. Ever.

---

## 2. GOVERNOR LIMITS — Know & Respect Them

| Resource       | Sync Limit | Async Limit |
|----------------|-----------|-------------|
| SOQL Queries   | 100       | 200         |
| DML Statements | 150       | 150         |
| CPU Time       | 10,000 ms | 60,000 ms   |
| Heap Size      | 6 MB      | 12 MB       |

- Use `Limits.getQueries()` / `Limits.getDmlStatements()` to monitor usage during development.
- Move heavy operations to async context (Batch, Queueable, @future).

---

## 3. ONE TRIGGER PER OBJECT — Handler Pattern

Every object gets exactly **one trigger** that delegates to a **handler class**.

```apex
// AccountTrigger.trigger — thin, no logic
trigger AccountTrigger on Account (before insert, before update, after insert, after update) {
    AccountTriggerHandler handler = new AccountTriggerHandler();
    if (Trigger.isBefore) {
        if (Trigger.isInsert) handler.beforeInsert(Trigger.new);
        if (Trigger.isUpdate) handler.beforeUpdate(Trigger.oldMap, Trigger.newMap);
    }
    if (Trigger.isAfter) {
        if (Trigger.isInsert) handler.afterInsert(Trigger.new);
        if (Trigger.isUpdate) handler.afterUpdate(Trigger.oldMap, Trigger.newMap);
    }
}
```

**Handler delegates to Domain/Service layer — no business logic in the trigger file itself.**

---

## 4. RECURSION PREVENTION

Always guard against trigger re-entry using a static counter.

```apex
public class TriggerHelper {
    private static Integer executionCount = 0;
    private static final Integer MAX_EXECUTIONS = 2;

    public static Boolean shouldExecute() {
        if (executionCount >= MAX_EXECUTIONS) return false;
        executionCount++;
        return true;
    }
}
```

---

## 5. ORDER OF EXECUTION (Know This for Design Decisions)

When a DML operation fires, Salesforce executes in this order:
1. Load record / system validation
2. Before-save Flows
3. **Before Triggers**
4. Validation Rules & Duplicate Rules
5. Save to DB (not committed)
6. **After Triggers**
7. Assignment / Auto-response / Workflow Rules
8. Workflow field updates → re-fires before+after triggers **once**
9. Process Builder / Flows
10. Roll-up Summary recalculation
11. **COMMIT** to database
12. Post-commit logic (email, Platform Events)

> ⚠️ Platform Events published in after triggers are **rolled back** if the transaction fails. Emails sent post-commit are **NOT** rolled back.

---

## 6. ASYNC PATTERNS — Choose the Right Tool

| Need                        | Use              |
|-----------------------------|------------------|
| Simple callout from trigger | `@future`        |
| Chaining jobs, complex args | `Queueable`      |
| 50k+ records                | `Batch Apex`     |
| Recurring scheduled jobs    | `Scheduled Apex` |

### @future — Simple fire-and-forget
```apex
@future(callout=true)
public static void makeCalloutAsync(Set<Id> recordIds) {
    // callouts, mixed DML, avoid in bulk scenarios
}
```
> ❌ Cannot chain. ❌ Only primitive arguments. ❌ No job monitoring.

### Queueable — Chainable async with object args
```apex
public class MyQueueable implements Queueable, Database.AllowsCallouts {
    private List<SObject> records;
    public MyQueueable(List<SObject> records) { this.records = records; }

    public void execute(QueueableContext ctx) {
        // process records
        if (moreToProcess) System.enqueueJob(new MyQueueable(nextBatch));
    }
}
System.enqueueJob(new MyQueueable(records));
```

### Batch Apex — Large datasets with Stateful tracking
```apex
global class MyBatch implements Database.Batchable<SObject>, Database.Stateful {
    global Integer processed = 0;

    global Database.QueryLocator start(Database.BatchableContext bc) {
        return Database.getQueryLocator([SELECT Id FROM Account WHERE ...]);
    }

    global void execute(Database.BatchableContext bc, List<Account> scope) {
        // process scope (default 200 records)
        processed += scope.size();
    }

    global void finish(Database.BatchableContext bc) {
        // send report, chain next batch if needed
    }
}
Database.executeBatch(new MyBatch(), 200);
```

**Batch limits:** Max 5 active jobs. Max 100 in Flex Queue. Max 50M records via QueryLocator.

---

## 7. SOQL BEST PRACTICES

### Use Relationship Queries to Reduce Query Count
```apex
// Child-to-Parent (up to 5 levels)
SELECT Id, Name, Account.Name, Account.Owner.Name
FROM Contact WHERE Account.Industry = 'Technology'

// Parent-to-Child (1 level only)
SELECT Id, Name,
    (SELECT Id, FirstName FROM Contacts),
    (SELECT Id, StageName FROM Opportunities WHERE IsClosed = false)
FROM Account WHERE Id IN :accountIds
```

### Write Selective Queries (avoid full table scans)
A query is **selective** if it returns < 10% of total records OR < 333,333 records.

```apex
// ❌ Non-selective — no index on Description
SELECT Id FROM Account WHERE Description LIKE '%keyword%'

// ✅ Selective — filter on indexed field first
SELECT Id FROM Account WHERE CreatedDate = LAST_N_DAYS:30 AND Description LIKE '%keyword%'
```

**Indexed fields by default:** `Id`, `OwnerId`, `RecordTypeId`, `CreatedDate`, `LastModifiedDate`, External ID fields, Unique fields, Lookup/MD fields.

### SOQL For Loops for Large Datasets
```apex
// Processes 200 records at a time, avoids heap overflow
for (Account[] batch : [SELECT Id, Name FROM Account WHERE Industry = 'Tech']) {
    // process batch, memory released after each chunk
}
```

---

## 8. SECURITY — Always Enforce

### Sharing Keywords
```apex
public with sharing class AccountController { }      // Enforces record-level security ✅
public without sharing class AdminService { }        // Bypasses sharing — system ops only
public inherited sharing class FlexService { }       // Inherits caller's sharing context
```

### FLS / CRUD Enforcement
```apex
// User Mode — auto-enforces CRUD + FLS (API 58.0+)
List<Account> accounts = [SELECT Id, Name FROM Account WITH USER_MODE];
Database.insert(records, AccessLevel.USER_MODE);

// stripInaccessible — remove fields user can't see
SObjectAccessDecision decision = Security.stripInaccessible(AccessType.READABLE, accounts);
return decision.getRecords();
```

> Default class mode is **System Mode** (bypasses CRUD/FLS). Always be intentional.

---

## 9. ENTERPRISE DESIGN PATTERNS

### Service Layer — Encapsulate Business Logic
```apex
public class AccountService {
    public static List<Account> createAccounts(List<Account> accounts) {
        validateAccounts(accounts);
        setDefaultValues(accounts);
        insert accounts;
        createDefaultContacts(accounts);
        return accounts;
    }
    // private helpers below...
}
```

### Repository Pattern — Centralize SOQL
```apex
public class AccountRepository {
    public List<Account> findByIndustry(String industry) {
        return [SELECT Id, Name FROM Account WHERE Industry = :industry];
    }
    public List<Account> findWithContacts(Set<Id> ids) {
        return [SELECT Id, Name, (SELECT Id, FirstName FROM Contacts) FROM Account WHERE Id IN :ids];
    }
}
```

### Singleton Pattern — Cache Expensive Queries
```apex
public class RecordTypeManager {
    private static RecordTypeManager instance;
    private Map<String, Id> cache = new Map<String, Id>();

    private RecordTypeManager() {
        for (RecordType rt : [SELECT Id, DeveloperName, SObjectType FROM RecordType]) {
            cache.put(rt.SObjectType + '.' + rt.DeveloperName, rt.Id);
        }
    }

    public static RecordTypeManager getInstance() {
        if (instance == null) instance = new RecordTypeManager();
        return instance;
    }

    public Id getRecordTypeId(String obj, String devName) {
        return cache.get(obj + '.' + devName);
    }
}
// Usage — single SOQL no matter how many times called
Id rtId = RecordTypeManager.getInstance().getRecordTypeId('Account', 'Enterprise');
```

### Strategy Pattern — Runtime Algorithm Selection
```apex
public interface IDiscountStrategy { Decimal calculate(Opportunity opp); }
public class VIPDiscount implements IDiscountStrategy {
    public Decimal calculate(Opportunity opp) { return opp.Amount * 0.15; }
}
public class VolumeDiscount implements IDiscountStrategy {
    public Decimal calculate(Opportunity opp) { return opp.Amount > 100000 ? opp.Amount * 0.20 : 0.10; }
}
// Select at runtime based on data
IDiscountStrategy strat = opp.Account.Tier__c == 'VIP' ? new VIPDiscount() : new VolumeDiscount();
opp.Discount__c = strat.calculate(opp);
```

### Bulk State Transition Pattern — Detect Field Changes Efficiently
```apex
public static void handleStageTransitions(Map<Id, Opportunity> oldMap, List<Opportunity> newList) {
    List<Opportunity> wonOpps = new List<Opportunity>();
    for (Opportunity opp : newList) {
        if (opp.StageName == 'Closed Won' && oldMap.get(opp.Id).StageName != 'Closed Won') {
            wonOpps.add(opp);
        }
    }
    if (!wonOpps.isEmpty()) processWonOpportunities(wonOpps);
}
```

---

## 10. NEVER HARDCODE IDs

```apex
// ❌ BAD
Id rtId = '012000000000000';

// ✅ GOOD — dynamic lookup
Id rtId = Schema.SObjectType.Account
    .getRecordTypeInfosByDeveloperName()
    .get('Enterprise').getRecordTypeId();

// ✅ BEST — cached via Singleton
Id rtId = RecordTypeManager.getInstance().getRecordTypeId('Account', 'Enterprise');
```

---

## 11. TESTING STANDARDS

- Minimum **75% code coverage** (aim for 90%+)
- Always test with **200 records** to verify bulkification
- Use `@TestSetup` for shared data (created once, rolled back per test)
- Test **negative scenarios** and error paths
- Wrap async calls between `Test.startTest()` / `Test.stopTest()`

```apex
@isTest
private class AccountServiceTest {

    @TestSetup
    static void setup() {
        List<Account> accs = new List<Account>();
        for (Integer i = 0; i < 200; i++) {
            accs.add(new Account(Name = 'Test ' + i, Industry = 'Technology'));
        }
        insert accs;
    }

    @isTest
    static void testBulkInsert_200Records() {
        Test.startTest();
        List<Account> accs = [SELECT Id FROM Account];
        for (Account a : accs) { a.Description = 'Bulk Test'; }
        update accs;
        Test.stopTest();

        System.assertEquals(200, [SELECT COUNT() FROM Account WHERE Description = 'Bulk Test'],
            'All 200 records should be updated');
    }

    @isTest
    static void testNegativeScenario_MissingName() {
        try {
            insert new Account(); // missing required fields
            System.assert(false, 'Should have thrown DmlException');
        } catch (DmlException e) {
            System.assert(e.getMessage().contains('REQUIRED_FIELD_MISSING'));
        }
    }
}
```

---

## 12. INTEGRATION PATTERNS

### Named Credentials — Never Hardcode Endpoints
```apex
HttpRequest req = new HttpRequest();
req.setEndpoint('callout:My_Named_Credential/api/v1/orders'); // ✅
req.setMethod('POST');
req.setHeader('Content-Type', 'application/json');
req.setTimeout(120000);
Http http = new Http();
HttpResponse res = http.send(req);
```

### Platform Events — Decoupled Architecture
```apex
// Publish
List<Order_Event__e> events = new List<Order_Event__e>();
events.add(new Order_Event__e(Order_Id__c = ord.Id, Status__c = ord.Status));
EventBus.publish(events);

// Subscribe (trigger on platform event)
trigger OrderEventTrigger on Order_Event__e (after insert) {
    for (Order_Event__e evt : Trigger.new) {
        // handle event — runs in its own transaction
    }
}
```

---

## CHECKLIST — Before Submitting Any Apex Class

- [ ] No SOQL inside `for` loops
- [ ] No DML inside `for` loops
- [ ] All methods accept `List<>` not single `SObject`
- [ ] Uses `with sharing` (unless intentionally system-level)
- [ ] No hardcoded Org IDs, Record Type IDs, or User IDs
- [ ] Test class covers 200-record bulk scenarios
- [ ] Async work uses appropriate pattern (Batch/Queueable/@future)
- [ ] Error handling present (try/catch, `Database.update(list, false)`)
- [ ] Separation of concerns: Trigger → Handler → Service → Repository

---

*Reference: Salesforce Apex Architecture & Interview Guide — Enterprise Implementation Patterns*
