# Testing C Data Structures

A data structure is not correct because its code looks reasonable.

It is correct only if every supported operation preserves its contract and invariant for every valid state, including states reached near boundaries and after failures.

Testing gives me executable evidence about that claim.

The word “executable” matters.

A written explanation says what I believe.

A test constructs a specific state, performs a specific operation, and checks whether the observed result agrees with the stated rule.

In C, that result is not limited to a returned value.

I may also need to check:

- which memory is owned
- which fields changed
- which fields did not change
- whether existing elements were preserved
- whether an output parameter remained untouched on failure
- whether an allocation was released
- whether a pointer became invalid after relocation
- whether an invariant still holds
- whether the program executed undefined behavior

This chapter builds a testing approach from those first principles.

It does not assume an external testing framework.

The complete examples use ordinary C17, a small test runner, strong compiler warnings, and dynamic-analysis tools.

## 1. Testing Is Executable Reasoning

Suppose I implement:

```c
bool int_buffer_push(IntBuffer *buffer, int value);
```

A weak test asks only:

> Did one call return `true`?

A serious test asks:

1. What state was the buffer in before the call?
2. Was growth required?
3. What must be true if the operation succeeds?
4. What must remain true if allocation fails?
5. Which old elements must retain their values?
6. Did logical size change exactly once?
7. Is capacity still at least size?
8. Can the structure still be destroyed safely?

The test is therefore a small proof attempt.

It does not prove the implementation for every possible execution.

It proves that one carefully constructed execution agrees with part of the specification.

Many tests combine evidence from many important executions.

The quality of that evidence depends more on the chosen states and observations than on the number of test functions.

One hundred tests that repeat the same easy path may be weaker than ten tests that cover distinct state transitions and failure modes.

## 2. What Tests Can and Cannot Establish

A passing test establishes:

> For this build, on this execution, with these inputs and this environment, every observation made by the test matched its expectation.

That is valuable.

It is not the same as:

> The program is correct for all possible inputs and machines.

Tests sample behavior.

They can reveal bugs by producing counterexamples.

One failing execution is enough to disprove a universal claim such as:

> Push always preserves existing elements.

Passing executions do not by themselves prove the universal claim.

Correctness evidence comes from several sources:

- precise contracts
- invariants
- code review
- mathematical reasoning
- compiler diagnostics
- deterministic tests
- randomized or generated tests
- sanitizers
- leak detection
- static analysis
- production observations

These sources complement one another.

A sanitizer can reveal an out-of-bounds write that output checks missed.

An invariant review can reveal an untested state that the sanitizer never executed.

A deterministic regression test can preserve the exact input that once triggered a bug.

Testing is strongest when it is part of a reasoning system rather than a ritual performed after coding.

## 3. The Specification Comes Before the Test

A test needs an expected result.

That expectation must come from a specification, not from copying the implementation's current behavior.

For a data structure, the specification has three useful layers.

### Operation Contract

The contract describes one public operation.

For example:

```txt
int_buffer_get(buffer, index, value_out)
```

Preconditions:

- `buffer` points to an initialized valid buffer
- `value_out` is non-null

Success condition:

- `index < buffer->size`

Success effects:

- returns `true`
- writes the element at `index` to `*value_out`
- does not mutate the buffer

Failure effects:

- returns `false`
- leaves `*value_out` unchanged
- does not mutate the buffer

Now the test has something precise to check.

Without the failure-effect rule, either of these implementations could appear acceptable:

```c
*value_out = 0;
return false;
```

or:

```c
return false;
```

The contract decides which behavior is correct.

### Structure Invariant

The invariant describes every valid observable state of the structure.

For an owning integer buffer:

```txt
size <= capacity
```

and:

```txt
capacity == 0  implies  data == NULL
capacity > 0   implies  data points to owned storage
```

and:

```txt
elements in [0, size) are initialized logical values
```

The invariant must hold:

- after initialization
- before every public operation
- after every successful operation
- after every failed operation whose contract promises preservation
- after destruction, if destruction leaves a reusable empty object

### State Transition

An operation moves the structure from one valid state to another.

For push without growth:

```txt
before:
    size = s
    capacity = c
    s < c

after success:
    size = s + 1
    capacity = c
    old elements unchanged
    new element stored at index s
```

For push requiring growth:

```txt
before:
    size = capacity = c

after success:
    size = c + 1
    capacity > c
    old elements unchanged
    new element stored at index c
```

For failed growth:

```txt
after failure:
    data unchanged
    size unchanged
    capacity unchanged
    old elements unchanged
    ownership unchanged
```

A test suite should cover transitions, not merely isolated values.

## 4. A Test Needs an Oracle

An oracle is the rule that tells the test what the correct result should be.

For simple operations, the oracle may be a literal:

```c
EXPECT_INT(context, actual, 42);
```

For a sort, the oracle may check two properties:

1. the result is ordered
2. the result contains the same multiset of elements as the input

For a stack, a simple reference model may be an ordinary array plus a size counter.

For a complicated operation, the oracle may be a slower implementation whose correctness is easier to understand.

A dangerous test uses the production algorithm to calculate its own expected answer.

For example:

```c
int expected = complicated_production_formula(input);
int actual = function_under_test(input);
EXPECT_INT(context, actual, expected);
```

If both calls use the same flawed logic, the test can pass while proving nothing new.

An oracle should be independent enough to fail when the implementation's assumption is wrong.

### Oracle Strength

This assertion is weak:

```c
EXPECT_TRUE(context, buffer.capacity >= 4);
```

It checks only one fact.

These observations are stronger:

```c
EXPECT_SIZE(context, buffer.size, 4);
EXPECT_TRUE(context, buffer.capacity >= buffer.size);
EXPECT_INT(context, buffer.data[0], 10);
EXPECT_INT(context, buffer.data[1], 20);
EXPECT_INT(context, buffer.data[2], 30);
EXPECT_INT(context, buffer.data[3], 40);
```

But more assertions are not automatically better.

If the public contract does not promise an exact capacity-growth factor, this is over-specific:

```c
EXPECT_SIZE(context, buffer.capacity, 8);
```

The test would reject a correct implementation that grows to `6`.

A good oracle is:

- strong enough to detect incorrect behavior
- limited to behavior the contract actually promises

## 5. Layers of Testing Evidence

No single layer sees every class of problem.

### Compiler Diagnostics

Warnings can expose:

- incorrect format strings
- suspicious conversions
- missing declarations
- unreachable code
- unused results
- accidental fallthrough

I should compile tests and production code with the same strict warning policy.

### Unit Tests

A unit test exercises a small public behavior in isolation.

Examples:

- initialize an empty buffer
- reject an out-of-range index
- push one element
- preserve output on failed get

### Sequence Tests

A data structure is stateful.

A sequence test performs several operations and checks the resulting history.

Examples:

- initialize, push, get, destroy
- push through several capacity changes
- clone, mutate one copy, compare both

### Failure-Path Tests

Failure paths are often less exercised than success paths.

Examples:

- allocation fails during first growth
- allocation fails after the buffer already contains elements
- cloning fails while the destination already owns data

### Property and Model Tests

These compare many generated operation sequences against a simpler reference model or general property.

### Dynamic Analysis

AddressSanitizer, UndefinedBehaviorSanitizer, and Valgrind can detect classes of invalid memory behavior that ordinary value assertions may not observe.

### Integration Tests

These test several components together.

For later chapters, an integration test might exercise a queue built on top of a dynamic array.

The testing pyramid is not a law.

The useful question is:

> Which layer can observe this particular risk with the clearest and cheapest evidence?

## 6. The System Under Test

To keep the chapter concrete, I will test an owning `IntBuffer`.

Its public shape is:

```c
typedef struct IntBuffer IntBuffer;

typedef struct {
    void *context;
    void *(*reallocate)(
        void *context,
        void *memory,
        size_t new_size
    );
    void (*deallocate)(void *context, void *memory);
} Allocator;

struct IntBuffer {
    int *data;
    size_t size;
    size_t capacity;
    Allocator allocator;
};
```

The allocator is explicit because allocation failure must be testable.

Ordinary execution uses wrappers around `realloc` and `free`.

Tests can supply a controlled allocator that fails on a chosen call and counts live allocations.

This is dependency injection in plain C.

The data structure does not know whether its allocator is:

- the system allocator
- a deterministic failing allocator
- a tracking allocator
- an arena
- another compatible implementation

It depends only on the function-pointer contract.

### Public Operations

The complete laboratory program will implement:

```c
Allocator system_allocator(void);

bool int_buffer_init(
    IntBuffer *buffer,
    Allocator allocator
);

void int_buffer_destroy(IntBuffer *buffer);

bool int_buffer_is_valid(const IntBuffer *buffer);

bool int_buffer_reserve(
    IntBuffer *buffer,
    size_t new_capacity
);

bool int_buffer_push(
    IntBuffer *buffer,
    int value
);

bool int_buffer_get(
    const IntBuffer *buffer,
    size_t index,
    int *value_out
);

bool int_buffer_clone(
    IntBuffer *destination,
    const IntBuffer *source
);
```

The chapter restates the implementation later.

That repetition is deliberate.

The testing chapter must remain runnable even if it is read separately from the memory chapter.

## 7. Define the State Model

Before writing cases, I classify the meaningful buffer states.

| State | `size` | `capacity` | `data` |
|---|---:|---:|---|
| Uninitialized | indeterminate | indeterminate | indeterminate |
| Valid empty, no allocation | `0` | `0` | `NULL` |
| Valid empty, reserved | `0` | greater than `0` | owned allocation |
| Partially full | between `1` and `capacity - 1` | greater than `size` | owned allocation |
| Full | equal to `capacity` | greater than `0` | owned allocation |
| Destroyed and reset | `0` | `0` | `NULL` |

The uninitialized state is not a valid input to public operations other than initialization.

The destroyed-and-reset state is intentionally the same observable state as valid empty.

Because destruction retains the allocator configuration, the object can be reused.

### Transition Map

```txt
uninitialized
      |
      | init
      v
empty, no allocation
      |
      | reserve or push
      v
empty reserved / partially full / full
      |                 |
      | push            | clone into destination
      v                 v
partially full / full   independent valid copy
      |
      | destroy
      v
empty, no allocation
```

Failure transitions add self-loops:

```txt
valid state
    |
    | allocation attempt fails
    v
same valid state
```

The phrase “same valid state” is stronger than “did not crash.”

It means the exact pre-failure data, size, capacity, and ownership remain usable.

## 8. Start With One Direct Test

Before building a harness, I can write the smallest executable check:

```c
#include <assert.h>

int main(void) {
    IntBuffer buffer;

    assert(int_buffer_init(&buffer, system_allocator()));
    assert(buffer.data == NULL);
    assert(buffer.size == 0);
    assert(buffer.capacity == 0);
    assert(int_buffer_is_valid(&buffer));

    int_buffer_destroy(&buffer);
    return 0;
}
```

This test has value.

It also has limitations.

If an assertion fails, the process aborts.

The diagnostic may show only the failed expression.

Later tests do not run.

There is no summary.

The behavior of standard `assert` changes when the program is compiled with:

```sh
-DNDEBUG
```

Every standard assertion expression is then removed.

That makes `assert` useful for programmer assumptions, but not a complete testing interface.

I will build a small harness that always evaluates test checks and reports failures without depending on `NDEBUG`.

## 9. Arrange, Act, Assert, Cleanup

A clear test usually has four phases.

### Arrange

Construct the starting state.

```c
IntBuffer buffer;
bool initialized = int_buffer_init(
    &buffer,
    system_allocator()
);
```

### Act

Perform the operation being tested.

```c
bool pushed = int_buffer_push(&buffer, 42);
```

### Assert

Check the promised observations.

```c
EXPECT_TRUE(context, pushed);
EXPECT_SIZE(context, buffer.size, 1);
EXPECT_INT(context, buffer.data[0], 42);
```

### Cleanup

Release every resource acquired during arrange or act.

```c
int_buffer_destroy(&buffer);
```

The complete shape is:

```c
static void test_push_one(TestContext *context) {
    IntBuffer buffer;

    bool initialized = int_buffer_init(
        &buffer,
        system_allocator()
    );

    REQUIRE_TRUE(context, initialized, cleanup);

    bool pushed = int_buffer_push(&buffer, 42);

    EXPECT_TRUE(context, pushed);

    if (pushed) {
        EXPECT_SIZE(context, buffer.size, 1);
        EXPECT_INT(context, buffer.data[0], 42);
        EXPECT_TRUE(context, int_buffer_is_valid(&buffer));
    }

cleanup:
    if (initialized) {
        int_buffer_destroy(&buffer);
    }
}
```

The `REQUIRE_TRUE` macro jumps to cleanup when a prerequisite fails.

That prevents two common problems:

- continuing with an invalid fixture and causing a misleading crash
- returning early and leaking test resources

This is one controlled use of `goto`.

In C cleanup code, a forward jump to one exit section is often clearer than duplicating release logic across several branches.

## 10. Assertions and Runtime Error Handling Are Different

Production code and test code make different promises.

### Runtime Validation

This checks input that can be wrong during ordinary execution:

```c
if (buffer == NULL) {
    return false;
}
```

The caller receives a defined failure result.

The check remains in release builds.

### Internal Assertion

This checks a condition the programmer believes must already be true:

```c
assert(buffer->size <= buffer->capacity);
```

If it fails, there is a programming error.

The standard assertion may disappear under `NDEBUG`.

### Test Expectation

This records whether observed behavior matches a test oracle:

```c
EXPECT_SIZE(context, buffer.size, expected_size);
```

It must not disappear in a release-like test build.

The three mechanisms should not be confused.

This is wrong:

```c
assert(int_buffer_push(&buffer, 42));
```

if the same source will ever be compiled with `NDEBUG`.

When assertions are disabled, the entire expression is not evaluated.

The push never happens.

A safer separation is:

```c
bool pushed = int_buffer_push(&buffer, 42);
assert(pushed);
```

The operation still executes when assertions are disabled.

In the chapter's test harness, `EXPECT_TRUE` always evaluates its expression.

## 11. Test Outcomes Need More Than Pass or Crash

A useful runner distinguishes:

- passed test
- failed expectation
- failed prerequisite
- process crash
- sanitizer failure
- build failure
- timeout, when a harness supports time limits

The small runner in this chapter records expectation failures and continues to the next test function.

It cannot recover from every process-level failure.

If a test:

- dereferences null
- executes an illegal instruction
- corrupts the stack
- loops forever

the whole test process may stop.

Larger test systems sometimes isolate cases in separate processes.

That is useful, but it is not required for the first testing foundation.

Here, sanitizer output and a focused test name will provide the failure context.

The runner returns a nonzero process status when any expectation fails.

That is essential for shell scripts, Makefiles, and continuous integration.

```txt
exit status 0     all tests passed
nonzero status    build or test failure
```

Printed green text without a meaningful exit status is not automation-ready.

## 12. Build a Minimal Harness From First Principles

The harness needs:

- a place to count checks and failures
- assertion helpers
- a test function type
- a name paired with each function
- a runner
- a process exit status

The core types are:

```c
typedef struct {
    size_t checks;
    size_t failures;
} TestContext;

typedef void (*TestFunction)(TestContext *context);

typedef struct {
    const char *name;
    TestFunction function;
} TestCase;
```

Each test receives the shared result counters.

It does not receive mutable state from another test.

The most basic check helper is:

```c
static bool test_expect_true(
    TestContext *context,
    bool condition,
    const char *expression,
    const char *file,
    int line
) {
    ++context->checks;

    if (condition) {
        return true;
    }

    ++context->failures;

    fprintf(
        stderr,
        "%s:%d: expectation failed: %s\n",
        file,
        line,
        expression
    );

    return false;
}
```

The macro captures source location and expression text:

```c
#define EXPECT_TRUE(context, expression)                         \
    ((void)test_expect_true(                                    \
        (context),                                              \
        (expression),                                           \
        #expression,                                            \
        __FILE__,                                               \
        __LINE__                                                \
    ))
```

The `#expression` operation stringifies the source tokens passed to the macro.

`__FILE__` and `__LINE__` identify the call site.

The helper receives the evaluated condition only once.

That last fact is important.

This macro is dangerous:

```c
#define BAD_EXPECT_EQUAL(actual, expected) \
    ((actual) == (expected) ? pass() : print(actual, expected))
```

`actual` may be evaluated once in the comparison and again while printing.

If the caller writes:

```c
BAD_EXPECT_EQUAL(values[index++], 10);
```

the test itself changes behavior by incrementing twice on failure.

Robust assertion helpers evaluate each value once and pass the results to a function.

## 13. Diagnostics Must Show the Difference

This failure:

```txt
expectation failed: buffer.size == expected
```

says what comparison failed.

This is better:

```txt
test_buffer.c:184: expected buffer.size == expected
    actual:   3
    expected: 4
```

The test runner therefore provides typed helpers:

```c
static bool test_expect_size(
    TestContext *context,
    size_t actual,
    size_t expected,
    const char *actual_text,
    const char *expected_text,
    const char *file,
    int line
);
```

Its macro passes already evaluated values:

```c
#define EXPECT_SIZE(context, actual, expected)                   \
    ((void)test_expect_size(                                    \
        (context),                                              \
        (actual),                                               \
        (expected),                                             \
        #actual,                                                \
        #expected,                                              \
        __FILE__,                                               \
        __LINE__                                                \
    ))
```

The function accepts values, not expressions.

Therefore each macro argument is evaluated once before the call.

Separate helpers are useful for:

- `bool`
- `int`
- `size_t`
- pointers
- strings
- byte ranges
- structure-specific logical equality

A single “everything is an integer” assertion creates unsafe conversions and unreadable diagnostics.

The assertion type should match the domain being compared.

## 14. Fatal and Nonfatal Checks

Some failures allow the test to continue.

```c
EXPECT_SIZE(context, buffer.size, 3);
EXPECT_TRUE(context, int_buffer_is_valid(&buffer));
```

Even if the first expectation fails, the second observation may still provide useful evidence.

Other checks establish prerequisites.

```c
bool initialized = int_buffer_init(
    &buffer,
    system_allocator()
);

REQUIRE_TRUE(context, initialized, cleanup);
```

If initialization failed, continuing to call buffer operations would test an invalid fixture rather than the intended behavior.

The chapter harness defines:

```c
#define REQUIRE_TRUE(context, expression, cleanup_label)         \
    do {                                                         \
        if (!test_expect_true(                                   \
                (context),                                      \
                (expression),                                   \
                #expression,                                    \
                __FILE__,                                       \
                __LINE__)) {                                    \
            goto cleanup_label;                                 \
        }                                                        \
    } while (0)
```

The `do { ... } while (0)` wrapper makes the multi-statement macro behave syntactically like one statement.

This works safely:

```c
if (condition) {
    REQUIRE_TRUE(context, initialized, cleanup);
}
```

Without the wrapper, macro expansion can interact badly with surrounding `if` and `else` statements.

Fatality belongs to the test's control flow, not necessarily to the importance of the behavior.

A failed prerequisite is fatal to that test because later observations would be invalid.

It does not need to terminate the entire test program.

## 15. Tests Must Be Independent

A test should not depend on another test having run first.

This is fragile:

```c
static IntBuffer shared;

static void test_initialize(TestContext *context) {
    EXPECT_TRUE(
        context,
        int_buffer_init(&shared, system_allocator())
    );
}

static void test_push(TestContext *context) {
    EXPECT_TRUE(context, int_buffer_push(&shared, 42));
}
```

`test_push` fails or becomes invalid when:

- run alone
- run before `test_initialize`
- initialization test exits early
- another test leaves `shared` dirty

Each test should arrange its own fixture:

```c
static void test_push(TestContext *context) {
    IntBuffer buffer;
    bool initialized = int_buffer_init(
        &buffer,
        system_allocator()
    );

    REQUIRE_TRUE(context, initialized, cleanup);
    EXPECT_TRUE(context, int_buffer_push(&buffer, 42));

cleanup:
    if (initialized) {
        int_buffer_destroy(&buffer);
    }
}
```

Independent tests can be:

- reordered
- filtered
- repeated
- debugged alone
- run in parallel by a more advanced runner

### Determinism

Given the same build and explicit test seed, a deterministic test should produce the same result.

Sources of accidental nondeterminism include:

- current time
- unseeded or hidden randomness
- thread scheduling
- filesystem ordering
- network responses
- shared global state
- uninitialized memory
- locale

If randomness is useful, the seed should be recorded and reproducible.

## 16. Table-Driven Tests Separate Data From Control

Several cases may share the same logic.

Suppose I want to test valid and invalid indices:

```c
typedef struct {
    size_t index;
    bool should_succeed;
    int expected_value;
} GetCase;

const GetCase cases[] = {
    {.index = 0, .should_succeed = true, .expected_value = 10},
    {.index = 1, .should_succeed = true, .expected_value = 20},
    {.index = 2, .should_succeed = false, .expected_value = 0},
    {.index = SIZE_MAX, .should_succeed = false, .expected_value = 0},
};
```

One loop executes the common arrangement and assertion pattern.

Table-driven tests make it easy to see:

- which cases exist
- which boundary is missing
- which expected result belongs to each input

They are not ideal when each case has a different state transition or cleanup requirement.

Clarity is the goal.

I should not force unrelated scenarios into one clever loop merely to reduce line count.

## 17. Test the Empty State First

Empty state is not a trivial afterthought.

It is the starting point and often the endpoint after cleanup.

For `IntBuffer`, initialization should establish:

```txt
data = NULL
size = 0
capacity = 0
allocator functions are valid
```

The first test should check:

- initialization succeeds with a valid allocator
- the invariant holds
- no allocation is performed
- a get at index `0` fails
- the get output remains unchanged
- destroy is safe
- destroy restores the same empty state

This catches errors such as:

- allocating unnecessarily during initialization
- leaving capacity uninitialized
- treating index zero as valid in an empty structure
- changing output on failure
- freeing an uninitialized pointer during destruction

## 18. Boundaries Are Transitions, Not Just Numbers

For an array-backed structure, important size boundaries include:

```txt
0
1
capacity - 1
capacity
capacity + 1
largest safely representable capacity
```

But the transition matters:

```txt
size = capacity - 1
push
size = capacity
```

does not require growth.

This transition:

```txt
size = capacity
push
size = capacity + 1
```

does require growth.

A useful sequence deliberately crosses several growth boundaries:

```txt
capacity: 0 -> 1 -> 2 -> 4 -> 8 -> ...
```

After every push, check:

- size increased by one
- capacity is at least size
- all earlier values remain
- the newly appended value is correct
- the invariant holds

I should not assert that the address always changes during growth.

`realloc` may extend a block in place.

I should not assert that the address never changes either.

The contract promises value preservation, not pointer identity.

## 19. Test Sequences Reveal Stateful Bugs

This test is useful:

```txt
push 10
get index 0
```

This sequence is stronger:

```txt
push 10
push 20
push 30
get 0
get 1
get 2
reject get 3
reserve 20
get 0
get 1
get 2
destroy
```

It checks that:

- multiple pushes compose correctly
- growth preserves order
- reserve preserves logical size
- reserve preserves values
- invalid get does not corrupt state
- cleanup works after a nontrivial history

Data-structure bugs often depend on history.

A field may be valid after one operation but inconsistent after a particular combination.

Sequence tests exercise that composition.

The expected state should still be checked at intermediate points.

If I check only the end, I may know that something went wrong without knowing which transition first violated the invariant.

## 20. Test Copy Independence, Not Only Equality

Immediately after deep cloning:

```txt
source values      = [10, 20, 30]
destination values = [10, 20, 30]
```

Value equality is necessary.

It is not sufficient.

A shallow copy also produces equal values at first.

The test must establish independence:

1. source and destination data pointers differ for nonempty buffers
2. mutating the destination does not change the source
3. destroying the source does not invalidate the destination
4. destroying the destination does not double-free the source

A deep-copy test should therefore perform a post-clone mutation:

```c
destination.data[0] = 999;

EXPECT_INT(context, source.data[0], 10);
EXPECT_INT(context, destination.data[0], 999);
```

Directly modifying the public `data` field is acceptable in this laboratory because the representation is visible.

A later opaque API would use a public setter.

The principle remains:

> Test the semantic property that distinguishes deep copy from shallow copy.

## 21. Failure Atomicity Must Be Observable

An operation is failure-atomic when failure leaves the original logical state unchanged.

For a push that needs allocation:

```txt
before:
    data = P
    size = 1
    capacity = 1
    values = [10]

allocation fails

after:
    data = P
    size = 1
    capacity = 1
    values = [10]
```

The test records a snapshot before the operation:

```c
int *old_data = buffer.data;
size_t old_size = buffer.size;
size_t old_capacity = buffer.capacity;
int old_first = buffer.data[0];
```

After forced failure, it checks every promised preservation property.

For clone failure, the destination may already own different data.

The test must prove that failed cloning does not:

- destroy the old destination
- replace its pointer
- change its size
- change its capacity
- change its values
- leak a partially created copy

Failure tests should end by using and destroying the preserved object.

That provides evidence that the state is not merely numerically unchanged but remains operationally valid.

## 22. Allocation Failure Needs a Test Seam

Waiting for the real machine to run out of memory is not a useful unit-testing strategy.

It is:

- nondeterministic
- potentially destructive to the environment
- difficult to target at one allocation
- too slow
- hard to reproduce

The design needs a seam where the test can substitute controlled behavior.

The chapter uses:

```c
typedef struct {
    void *context;
    void *(*reallocate)(
        void *context,
        void *memory,
        size_t new_size
    );
    void (*deallocate)(void *context, void *memory);
} Allocator;
```

Production supplies:

```c
realloc
free
```

through wrappers.

The test supplies a tracker:

```c
typedef struct {
    size_t reallocate_calls;
    size_t deallocate_calls;
    size_t live_blocks;
    size_t invalid_deallocations;
    size_t fail_on_call;
} TrackingAllocator;
```

The tracker can say:

```txt
fail_on_call = reallocate_calls + 1
```

The next allocation attempt then fails deterministically.

It also counts live blocks, making ownership expectations observable.

This technique is more useful than a “mock” that merely returns hard-coded values.

It models the behavior relevant to the contract:

- allocation success
- allocation failure
- deallocation
- outstanding ownership

## 23. Complete Runnable Testing Laboratory

The following program is one translation unit.

It contains:

- the allocator abstraction
- a complete `IntBuffer`
- the tracking allocator
- typed test expectations
- nine independent tests
- the test runner

Save it as:

```txt
testing_lab.c
```

Compile it using the commands later in this chapter.

```c
#include <stdbool.h>
#include <inttypes.h>
#include <stddef.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
    void *context;
    void *(*reallocate)(
        void *context,
        void *memory,
        size_t new_size
    );
    void (*deallocate)(void *context, void *memory);
} Allocator;

typedef struct {
    int *data;
    size_t size;
    size_t capacity;
    Allocator allocator;
} IntBuffer;

static void *system_reallocate(
    void *context,
    void *memory,
    size_t new_size
) {
    (void)context;
    return realloc(memory, new_size);
}

static void system_deallocate(void *context, void *memory) {
    (void)context;
    free(memory);
}

static Allocator system_allocator(void) {
    Allocator allocator = {
        .context = NULL,
        .reallocate = system_reallocate,
        .deallocate = system_deallocate,
    };

    return allocator;
}

static bool allocator_is_valid(Allocator allocator) {
    return allocator.reallocate != NULL
        && allocator.deallocate != NULL;
}

static bool int_buffer_is_valid(const IntBuffer *buffer) {
    if (buffer == NULL) {
        return false;
    }

    if (!allocator_is_valid(buffer->allocator)) {
        return false;
    }

    if (buffer->size > buffer->capacity) {
        return false;
    }

    if (buffer->capacity > SIZE_MAX / sizeof *buffer->data) {
        return false;
    }

    if (buffer->capacity == 0) {
        return buffer->data == NULL;
    }

    return buffer->data != NULL;
}

static bool int_buffer_init(
    IntBuffer *buffer,
    Allocator allocator
) {
    if (buffer == NULL || !allocator_is_valid(allocator)) {
        return false;
    }

    buffer->data = NULL;
    buffer->size = 0;
    buffer->capacity = 0;
    buffer->allocator = allocator;
    return true;
}

static void int_buffer_destroy(IntBuffer *buffer) {
    if (buffer == NULL) {
        return;
    }

    if (
        buffer->data != NULL
        && buffer->allocator.deallocate != NULL
    ) {
        buffer->allocator.deallocate(
            buffer->allocator.context,
            buffer->data
        );
    }

    /*
     * Keep the allocator so a properly initialized object remains
     * a valid, reusable empty buffer after destruction.
     */
    buffer->data = NULL;
    buffer->size = 0;
    buffer->capacity = 0;
}

static bool int_buffer_allocation_bytes(
    size_t capacity,
    size_t *bytes_out
) {
    if (bytes_out == NULL) {
        return false;
    }

    if (capacity > SIZE_MAX / sizeof(int)) {
        return false;
    }

    *bytes_out = capacity * sizeof(int);
    return true;
}

static bool int_buffer_reserve(
    IntBuffer *buffer,
    size_t new_capacity
) {
    if (!int_buffer_is_valid(buffer)) {
        return false;
    }

    if (new_capacity <= buffer->capacity) {
        return true;
    }

    size_t new_bytes;

    if (!int_buffer_allocation_bytes(
            new_capacity,
            &new_bytes)) {
        return false;
    }

    int *resized = buffer->allocator.reallocate(
        buffer->allocator.context,
        buffer->data,
        new_bytes
    );

    if (resized == NULL) {
        return false;
    }

    buffer->data = resized;
    buffer->capacity = new_capacity;
    return true;
}

static bool int_buffer_push(
    IntBuffer *buffer,
    int value
) {
    if (!int_buffer_is_valid(buffer)) {
        return false;
    }

    if (buffer->size == buffer->capacity) {
        size_t next_capacity;

        if (buffer->capacity == 0) {
            next_capacity = 1;
        } else {
            if (buffer->capacity > SIZE_MAX / 2) {
                return false;
            }

            next_capacity = buffer->capacity * 2;
        }

        if (!int_buffer_reserve(buffer, next_capacity)) {
            return false;
        }
    }

    buffer->data[buffer->size] = value;
    ++buffer->size;
    return true;
}

static bool int_buffer_get(
    const IntBuffer *buffer,
    size_t index,
    int *value_out
) {
    if (
        !int_buffer_is_valid(buffer)
        || value_out == NULL
        || index >= buffer->size
    ) {
        return false;
    }

    *value_out = buffer->data[index];
    return true;
}

static bool int_buffer_clone(
    IntBuffer *destination,
    const IntBuffer *source
) {
    if (
        !int_buffer_is_valid(destination)
        || !int_buffer_is_valid(source)
    ) {
        return false;
    }

    if (destination == source) {
        return true;
    }

    IntBuffer copy;

    if (!int_buffer_init(&copy, destination->allocator)) {
        return false;
    }

    if (source->size > 0) {
        if (!int_buffer_reserve(&copy, source->size)) {
            int_buffer_destroy(&copy);
            return false;
        }

        memcpy(
            copy.data,
            source->data,
            source->size * sizeof *source->data
        );

        copy.size = source->size;
    }

    int_buffer_destroy(destination);
    *destination = copy;
    return true;
}

typedef struct {
    size_t reallocate_calls;
    size_t deallocate_calls;
    size_t live_blocks;
    size_t invalid_deallocations;
    size_t fail_on_call;
} TrackingAllocator;

static void tracking_allocator_reset(
    TrackingAllocator *tracker
) {
    tracker->reallocate_calls = 0;
    tracker->deallocate_calls = 0;
    tracker->live_blocks = 0;
    tracker->invalid_deallocations = 0;
    tracker->fail_on_call = 0;
}

static void tracking_allocator_fail_next(
    TrackingAllocator *tracker
) {
    tracker->fail_on_call = tracker->reallocate_calls + 1;
}

static void *tracking_reallocate(
    void *context,
    void *memory,
    size_t new_size
) {
    TrackingAllocator *tracker = context;
    ++tracker->reallocate_calls;

    if (
        tracker->fail_on_call != 0
        && tracker->reallocate_calls == tracker->fail_on_call
    ) {
        return NULL;
    }

    bool creates_block = memory == NULL;
    void *resized = realloc(memory, new_size);

    if (resized != NULL && creates_block) {
        ++tracker->live_blocks;
    }

    return resized;
}

static void tracking_deallocate(
    void *context,
    void *memory
) {
    TrackingAllocator *tracker = context;

    if (memory == NULL) {
        return;
    }

    ++tracker->deallocate_calls;

    if (tracker->live_blocks == 0) {
        ++tracker->invalid_deallocations;
    } else {
        --tracker->live_blocks;
    }

    free(memory);
}

static Allocator tracking_allocator(
    TrackingAllocator *tracker
) {
    Allocator allocator = {
        .context = tracker,
        .reallocate = tracking_reallocate,
        .deallocate = tracking_deallocate,
    };

    return allocator;
}

typedef struct {
    size_t checks;
    size_t failures;
} TestContext;

typedef void (*TestFunction)(TestContext *context);

typedef struct {
    const char *name;
    TestFunction function;
} TestCase;

static bool test_expect_true(
    TestContext *context,
    bool condition,
    const char *expression,
    const char *file,
    int line
) {
    ++context->checks;

    if (condition) {
        return true;
    }

    ++context->failures;

    fprintf(
        stderr,
        "%s:%d: expectation failed: %s\n",
        file,
        line,
        expression
    );

    return false;
}

static bool test_expect_size(
    TestContext *context,
    size_t actual,
    size_t expected,
    const char *actual_text,
    const char *expected_text,
    const char *file,
    int line
) {
    ++context->checks;

    if (actual == expected) {
        return true;
    }

    ++context->failures;

    fprintf(
        stderr,
        "%s:%d: expected %s == %s\n"
        "    actual:   %zu\n"
        "    expected: %zu\n",
        file,
        line,
        actual_text,
        expected_text,
        actual,
        expected
    );

    return false;
}

static bool test_expect_int(
    TestContext *context,
    int actual,
    int expected,
    const char *actual_text,
    const char *expected_text,
    const char *file,
    int line
) {
    ++context->checks;

    if (actual == expected) {
        return true;
    }

    ++context->failures;

    fprintf(
        stderr,
        "%s:%d: expected %s == %s\n"
        "    actual:   %d\n"
        "    expected: %d\n",
        file,
        line,
        actual_text,
        expected_text,
        actual,
        expected
    );

    return false;
}

static bool test_expect_pointer(
    TestContext *context,
    const void *actual,
    const void *expected,
    bool should_equal,
    const char *actual_text,
    const char *expected_text,
    const char *file,
    int line
) {
    ++context->checks;

    bool equal = actual == expected;

    if (equal == should_equal) {
        return true;
    }

    ++context->failures;

    fprintf(
        stderr,
        "%s:%d: expected %s %s %s\n"
        "    actual:   %p\n"
        "    expected: %p\n",
        file,
        line,
        actual_text,
        should_equal ? "==" : "!=",
        expected_text,
        actual,
        expected
    );

    return false;
}

#define EXPECT_TRUE(context, expression)                         \
    ((void)test_expect_true(                                    \
        (context),                                              \
        (expression),                                           \
        #expression,                                            \
        __FILE__,                                               \
        __LINE__                                                \
    ))

#define EXPECT_FALSE(context, expression)                        \
    EXPECT_TRUE((context), !(expression))

#define EXPECT_SIZE(context, actual, expected)                   \
    ((void)test_expect_size(                                    \
        (context),                                              \
        (actual),                                               \
        (expected),                                             \
        #actual,                                                \
        #expected,                                              \
        __FILE__,                                               \
        __LINE__                                                \
    ))

#define EXPECT_INT(context, actual, expected)                    \
    ((void)test_expect_int(                                     \
        (context),                                              \
        (actual),                                               \
        (expected),                                             \
        #actual,                                                \
        #expected,                                              \
        __FILE__,                                               \
        __LINE__                                                \
    ))

#define EXPECT_POINTER_EQUAL(context, actual, expected)          \
    ((void)test_expect_pointer(                                 \
        (context),                                              \
        (actual),                                               \
        (expected),                                             \
        true,                                                   \
        #actual,                                                \
        #expected,                                              \
        __FILE__,                                               \
        __LINE__                                                \
    ))

#define EXPECT_POINTER_NOT_EQUAL(context, actual, expected)      \
    ((void)test_expect_pointer(                                 \
        (context),                                              \
        (actual),                                               \
        (expected),                                             \
        false,                                                  \
        #actual,                                                \
        #expected,                                              \
        __FILE__,                                               \
        __LINE__                                                \
    ))

#define REQUIRE_TRUE(context, expression, cleanup_label)         \
    do {                                                        \
        if (!test_expect_true(                                  \
                (context),                                     \
                (expression),                                  \
                #expression,                                   \
                __FILE__,                                      \
                __LINE__)) {                                   \
            goto cleanup_label;                                \
        }                                                       \
    } while (0)

static void test_initialize_empty(TestContext *context) {
    TrackingAllocator tracker;
    tracking_allocator_reset(&tracker);

    IntBuffer buffer;
    bool initialized = int_buffer_init(
        &buffer,
        tracking_allocator(&tracker)
    );

    REQUIRE_TRUE(context, initialized, cleanup);

    EXPECT_TRUE(context, int_buffer_is_valid(&buffer));
    EXPECT_POINTER_EQUAL(context, buffer.data, NULL);
    EXPECT_SIZE(context, buffer.size, 0);
    EXPECT_SIZE(context, buffer.capacity, 0);
    EXPECT_SIZE(context, tracker.reallocate_calls, 0);
    EXPECT_SIZE(context, tracker.live_blocks, 0);

cleanup:
    if (initialized) {
        int_buffer_destroy(&buffer);
    }

    EXPECT_SIZE(context, tracker.live_blocks, 0);
    EXPECT_SIZE(context, tracker.invalid_deallocations, 0);
}

static void test_get_boundaries_preserve_output(
    TestContext *context
) {
    IntBuffer buffer;
    bool initialized = int_buffer_init(
        &buffer,
        system_allocator()
    );

    REQUIRE_TRUE(context, initialized, cleanup);
    REQUIRE_TRUE(
        context,
        int_buffer_push(&buffer, 10),
        cleanup
    );
    REQUIRE_TRUE(
        context,
        int_buffer_push(&buffer, 20),
        cleanup
    );

    int output = -777;

    EXPECT_TRUE(
        context,
        int_buffer_get(&buffer, 0, &output)
    );
    EXPECT_INT(context, output, 10);

    output = -777;
    EXPECT_TRUE(
        context,
        int_buffer_get(&buffer, 1, &output)
    );
    EXPECT_INT(context, output, 20);

    output = -777;
    EXPECT_FALSE(
        context,
        int_buffer_get(&buffer, 2, &output)
    );
    EXPECT_INT(context, output, -777);

    EXPECT_FALSE(
        context,
        int_buffer_get(&buffer, 0, NULL)
    );
    EXPECT_TRUE(context, int_buffer_is_valid(&buffer));

cleanup:
    if (initialized) {
        int_buffer_destroy(&buffer);
    }
}

static void test_push_across_growth_boundaries(
    TestContext *context
) {
    IntBuffer buffer;
    bool initialized = int_buffer_init(
        &buffer,
        system_allocator()
    );

    REQUIRE_TRUE(context, initialized, cleanup);

    for (size_t i = 0; i < 100; ++i) {
        bool pushed = int_buffer_push(
            &buffer,
            (int)(i * 3)
        );

        REQUIRE_TRUE(context, pushed, cleanup);

        EXPECT_SIZE(context, buffer.size, i + 1);
        EXPECT_TRUE(
            context,
            buffer.capacity >= buffer.size
        );
        EXPECT_TRUE(context, int_buffer_is_valid(&buffer));

        for (size_t j = 0; j <= i; ++j) {
            EXPECT_INT(
                context,
                buffer.data[j],
                (int)(j * 3)
            );
        }
    }

cleanup:
    if (initialized) {
        int_buffer_destroy(&buffer);
    }
}

static void test_reserve_preserves_values_and_never_shrinks(
    TestContext *context
) {
    IntBuffer buffer;
    bool initialized = int_buffer_init(
        &buffer,
        system_allocator()
    );

    REQUIRE_TRUE(context, initialized, cleanup);
    REQUIRE_TRUE(
        context,
        int_buffer_push(&buffer, 7),
        cleanup
    );
    REQUIRE_TRUE(
        context,
        int_buffer_push(&buffer, 11),
        cleanup
    );

    size_t old_size = buffer.size;

    REQUIRE_TRUE(
        context,
        int_buffer_reserve(&buffer, 32),
        cleanup
    );

    EXPECT_SIZE(context, buffer.size, old_size);
    EXPECT_TRUE(context, buffer.capacity >= 32);
    EXPECT_INT(context, buffer.data[0], 7);
    EXPECT_INT(context, buffer.data[1], 11);

    size_t grown_capacity = buffer.capacity;

    EXPECT_TRUE(
        context,
        int_buffer_reserve(&buffer, 1)
    );
    EXPECT_SIZE(context, buffer.capacity, grown_capacity);
    EXPECT_SIZE(context, buffer.size, old_size);
    EXPECT_TRUE(context, int_buffer_is_valid(&buffer));

cleanup:
    if (initialized) {
        int_buffer_destroy(&buffer);
    }
}

static void test_clone_is_independent(TestContext *context) {
    IntBuffer source;
    IntBuffer destination;

    bool source_initialized = int_buffer_init(
        &source,
        system_allocator()
    );
    bool destination_initialized = int_buffer_init(
        &destination,
        system_allocator()
    );

    REQUIRE_TRUE(
        context,
        source_initialized && destination_initialized,
        cleanup
    );

    REQUIRE_TRUE(
        context,
        int_buffer_push(&source, 10),
        cleanup
    );
    REQUIRE_TRUE(
        context,
        int_buffer_push(&source, 20),
        cleanup
    );
    REQUIRE_TRUE(
        context,
        int_buffer_push(&destination, -1),
        cleanup
    );

    REQUIRE_TRUE(
        context,
        int_buffer_clone(&destination, &source),
        cleanup
    );

    EXPECT_SIZE(context, destination.size, source.size);
    EXPECT_POINTER_NOT_EQUAL(
        context,
        destination.data,
        source.data
    );
    EXPECT_INT(context, destination.data[0], 10);
    EXPECT_INT(context, destination.data[1], 20);

    destination.data[0] = 999;

    EXPECT_INT(context, source.data[0], 10);
    EXPECT_INT(context, destination.data[0], 999);

    int_buffer_destroy(&source);
    source_initialized = false;

    EXPECT_INT(context, destination.data[1], 20);
    EXPECT_TRUE(
        context,
        int_buffer_is_valid(&destination)
    );

cleanup:
    if (source_initialized) {
        int_buffer_destroy(&source);
    }

    if (destination_initialized) {
        int_buffer_destroy(&destination);
    }
}

static void test_push_failure_preserves_state(
    TestContext *context
) {
    TrackingAllocator tracker;
    tracking_allocator_reset(&tracker);

    IntBuffer buffer;
    bool initialized = int_buffer_init(
        &buffer,
        tracking_allocator(&tracker)
    );

    REQUIRE_TRUE(context, initialized, cleanup);
    REQUIRE_TRUE(
        context,
        int_buffer_push(&buffer, 10),
        cleanup
    );

    int *old_data = buffer.data;
    size_t old_size = buffer.size;
    size_t old_capacity = buffer.capacity;
    int old_value = buffer.data[0];
    size_t old_live_blocks = tracker.live_blocks;

    tracking_allocator_fail_next(&tracker);

    EXPECT_FALSE(
        context,
        int_buffer_push(&buffer, 20)
    );

    EXPECT_POINTER_EQUAL(context, buffer.data, old_data);
    EXPECT_SIZE(context, buffer.size, old_size);
    EXPECT_SIZE(context, buffer.capacity, old_capacity);
    EXPECT_INT(context, buffer.data[0], old_value);
    EXPECT_SIZE(
        context,
        tracker.live_blocks,
        old_live_blocks
    );
    EXPECT_TRUE(context, int_buffer_is_valid(&buffer));

cleanup:
    if (initialized) {
        int_buffer_destroy(&buffer);
    }

    EXPECT_SIZE(context, tracker.live_blocks, 0);
    EXPECT_SIZE(context, tracker.invalid_deallocations, 0);
}

static void test_clone_failure_preserves_destination(
    TestContext *context
) {
    TrackingAllocator tracker;
    tracking_allocator_reset(&tracker);

    IntBuffer source;
    IntBuffer destination;

    bool source_initialized = int_buffer_init(
        &source,
        system_allocator()
    );
    bool destination_initialized = int_buffer_init(
        &destination,
        tracking_allocator(&tracker)
    );

    REQUIRE_TRUE(
        context,
        source_initialized && destination_initialized,
        cleanup
    );

    REQUIRE_TRUE(
        context,
        int_buffer_push(&source, 1),
        cleanup
    );
    REQUIRE_TRUE(
        context,
        int_buffer_push(&source, 2),
        cleanup
    );
    REQUIRE_TRUE(
        context,
        int_buffer_push(&destination, 99),
        cleanup
    );

    int *old_data = destination.data;
    size_t old_size = destination.size;
    size_t old_capacity = destination.capacity;
    int old_value = destination.data[0];
    size_t old_live_blocks = tracker.live_blocks;

    tracking_allocator_fail_next(&tracker);

    EXPECT_FALSE(
        context,
        int_buffer_clone(&destination, &source)
    );

    EXPECT_POINTER_EQUAL(
        context,
        destination.data,
        old_data
    );
    EXPECT_SIZE(context, destination.size, old_size);
    EXPECT_SIZE(
        context,
        destination.capacity,
        old_capacity
    );
    EXPECT_INT(context, destination.data[0], old_value);
    EXPECT_SIZE(
        context,
        tracker.live_blocks,
        old_live_blocks
    );
    EXPECT_TRUE(
        context,
        int_buffer_is_valid(&destination)
    );
    EXPECT_INT(context, source.data[0], 1);
    EXPECT_INT(context, source.data[1], 2);

cleanup:
    if (source_initialized) {
        int_buffer_destroy(&source);
    }

    if (destination_initialized) {
        int_buffer_destroy(&destination);
    }

    EXPECT_SIZE(context, tracker.live_blocks, 0);
    EXPECT_SIZE(context, tracker.invalid_deallocations, 0);
}

static void test_destroy_releases_and_resets(
    TestContext *context
) {
    TrackingAllocator tracker;
    tracking_allocator_reset(&tracker);

    IntBuffer buffer;
    bool initialized = int_buffer_init(
        &buffer,
        tracking_allocator(&tracker)
    );

    REQUIRE_TRUE(context, initialized, cleanup);
    REQUIRE_TRUE(
        context,
        int_buffer_push(&buffer, 42),
        cleanup
    );

    EXPECT_SIZE(context, tracker.live_blocks, 1);

    int_buffer_destroy(&buffer);

    EXPECT_SIZE(context, tracker.live_blocks, 0);
    EXPECT_SIZE(context, tracker.deallocate_calls, 1);
    EXPECT_POINTER_EQUAL(context, buffer.data, NULL);
    EXPECT_SIZE(context, buffer.size, 0);
    EXPECT_SIZE(context, buffer.capacity, 0);
    EXPECT_TRUE(context, int_buffer_is_valid(&buffer));

    int_buffer_destroy(&buffer);

    EXPECT_SIZE(context, tracker.deallocate_calls, 1);
    EXPECT_SIZE(context, tracker.invalid_deallocations, 0);

    REQUIRE_TRUE(
        context,
        int_buffer_push(&buffer, 77),
        cleanup
    );
    EXPECT_INT(context, buffer.data[0], 77);

cleanup:
    if (initialized) {
        int_buffer_destroy(&buffer);
    }

    EXPECT_SIZE(context, tracker.live_blocks, 0);
    EXPECT_SIZE(context, tracker.invalid_deallocations, 0);
}

static void test_invalid_arguments_are_rejected(
    TestContext *context
) {
    Allocator invalid_allocator = {
        .context = NULL,
        .reallocate = NULL,
        .deallocate = NULL,
    };

    IntBuffer buffer;

    EXPECT_FALSE(
        context,
        int_buffer_init(NULL, system_allocator())
    );
    EXPECT_FALSE(
        context,
        int_buffer_init(&buffer, invalid_allocator)
    );
    EXPECT_FALSE(context, int_buffer_is_valid(NULL));
    EXPECT_FALSE(context, int_buffer_push(NULL, 1));
    EXPECT_FALSE(context, int_buffer_reserve(NULL, 10));
    EXPECT_FALSE(context, int_buffer_get(NULL, 0, NULL));
    EXPECT_FALSE(context, int_buffer_clone(NULL, NULL));

    int_buffer_destroy(NULL);
}

static bool run_test(
    TestContext *context,
    const TestCase *test
) {
    size_t failures_before = context->failures;

    printf("[ RUN      ] %s\n", test->name);
    test->function(context);

    if (context->failures == failures_before) {
        printf("[       OK ] %s\n", test->name);
        return true;
    }

    printf("[  FAILED  ] %s\n", test->name);
    return false;
}

int main(void) {
    const TestCase tests[] = {
        {
            .name = "initialize empty",
            .function = test_initialize_empty,
        },
        {
            .name = "get boundaries preserve output",
            .function = test_get_boundaries_preserve_output,
        },
        {
            .name = "push across growth boundaries",
            .function = test_push_across_growth_boundaries,
        },
        {
            .name = "reserve preserves values",
            .function =
                test_reserve_preserves_values_and_never_shrinks,
        },
        {
            .name = "clone is independent",
            .function = test_clone_is_independent,
        },
        {
            .name = "push failure preserves state",
            .function = test_push_failure_preserves_state,
        },
        {
            .name = "clone failure preserves destination",
            .function =
                test_clone_failure_preserves_destination,
        },
        {
            .name = "destroy releases and resets",
            .function = test_destroy_releases_and_resets,
        },
        {
            .name = "invalid arguments are rejected",
            .function = test_invalid_arguments_are_rejected,
        },
    };

    TestContext context = {
        .checks = 0,
        .failures = 0,
    };

    size_t failed_tests = 0;
    size_t test_count = sizeof tests / sizeof tests[0];

    for (size_t i = 0; i < test_count; ++i) {
        if (!run_test(&context, &tests[i])) {
            ++failed_tests;
        }
    }

    printf(
        "\n%zu tests, %zu checks, %zu failed tests, "
        "%zu failed checks\n",
        test_count,
        context.checks,
        failed_tests,
        context.failures
    );

    return context.failures == 0
        ? EXIT_SUCCESS
        : EXIT_FAILURE;
}
```

The program is intentionally more verbose than a framework-based equivalent.

The verbosity exposes the mechanics a framework normally hides:

- function registration
- assertion dispatch
- source-location reporting
- prerequisite control flow
- fixture cleanup
- failure counting
- process status

Once these mechanics are understood, adopting a mature C testing framework becomes a tooling choice rather than magic.

## 24. Dry Run the Test Runner

Suppose the runner begins with:

```txt
checks = 0
failures = 0
failed_tests = 0
```

The first test is:

```txt
initialize empty
```

Before calling it, `run_test` records:

```txt
failures_before = 0
```

The test performs nine successful checks.

After it returns:

```txt
checks = 9
failures = 0
```

Because:

```txt
failures == failures_before
```

the runner prints:

```txt
[       OK ] initialize empty
```

Now imagine the next test contains:

```c
EXPECT_SIZE(context, buffer.size, 3);
```

but the actual size is `2`.

The helper:

1. increments `checks`
2. compares `2` with `3`
3. increments `failures`
4. prints the file, line, expression, actual value, and expected value
5. returns `false`

The test continues unless it used `REQUIRE_TRUE`.

When the test returns:

```txt
failures > failures_before
```

so the runner increments `failed_tests`.

At the end:

```c
return context.failures == 0
    ? EXIT_SUCCESS
    : EXIT_FAILURE;
```

The shell can observe the result:

```sh
./testing_lab
echo $?
```

On success:

```txt
0
```

On any failed expectation:

```txt
nonzero
```

This is how a human-readable test becomes a machine-enforceable gate.

## 25. Dry Run an Injected Allocation Failure

The push-failure test starts with:

```txt
tracker:
    reallocate_calls = 0
    live_blocks = 0
    fail_on_call = 0

buffer:
    data = NULL
    size = 0
    capacity = 0
```

The first push of `10` requires capacity `1`.

The tracking allocator receives:

```txt
memory = NULL
new_size = sizeof(int)
```

It increments:

```txt
reallocate_calls = 1
```

The real `realloc` succeeds.

Because the old pointer was null, a new live block was created:

```txt
live_blocks = 1
```

The buffer becomes:

```txt
data = P
size = 1
capacity = 1
values = [10]
```

The test calls:

```c
tracking_allocator_fail_next(&tracker);
```

This sets:

```txt
fail_on_call = reallocate_calls + 1 = 2
```

The next push requires growth.

The allocator increments:

```txt
reallocate_calls = 2
```

Because this equals `fail_on_call`, it returns null without calling the real allocator.

`int_buffer_reserve` returns `false` before changing `data` or `capacity`.

`int_buffer_push` returns `false` before writing the value or incrementing size.

The state remains:

```txt
data = P
size = 1
capacity = 1
values = [10]
live_blocks = 1
```

Finally, destruction calls the tracking deallocator:

```txt
deallocate_calls = 1
live_blocks = 0
```

The test has exercised a failure that would be difficult to trigger reliably with the system allocator alone.

It has also shown that the failure path preserves ownership.

## 26. Black-Box and White-Box Tests

A black-box test uses only the public interface.

For `IntBuffer`, it might:

- initialize
- push
- get
- clone
- destroy

It does not inspect `data`, `size`, or `capacity` directly.

A white-box test knows the representation.

The laboratory checks:

```c
buffer.size
buffer.capacity
buffer.data
```

and calls:

```c
int_buffer_is_valid(&buffer)
```

### Strength of Black-Box Tests

Black-box tests survive implementation changes.

If the buffer later uses:

- segmented storage
- small-buffer optimization
- a different growth rule

tests based only on public behavior may remain valid.

### Strength of White-Box Tests

White-box tests can localize invariant failures and verify implementation-specific promises.

They can directly inspect:

- capacity transitions
- pointer replacement
- internal counters
- ownership state

### The Tradeoff

Testing every private detail freezes the implementation.

Testing only final output may miss a broken invariant that happens not to affect the chosen example.

A practical split is:

- most tests exercise public contracts
- targeted internal tests validate difficult invariants
- representation-specific expectations are clearly labeled

An invariant checker can be compiled only in debug and test builds if exposing it publicly would weaken the API.

## 27. Properties Generalize Examples

An example says:

```txt
after pushing 10, 20, 30, get(1) returns 20
```

A property says:

> After successfully pushing a sequence of `n` values, the buffer size is `n`, and getting each valid index returns the corresponding pushed value.

Properties identify families of behavior.

Useful `IntBuffer` properties include:

### Size Property

After every successful push:

```txt
new size = old size + 1
```

### Prefix Preservation Property

After successful reserve or push:

```txt
every old logical element retains its value and index
```

### Capacity Property

For every valid state:

```txt
capacity >= size
```

### Get Property

For every:

```txt
0 <= index < size
```

`get` succeeds and returns the model value.

For:

```txt
index >= size
```

`get` fails and preserves the output.

### Clone Property

After successful clone:

- logical values are equal
- the owners are independent
- changing one does not change the other

### Failure Preservation Property

If an allocating operation fails:

```txt
observable logical state after == observable logical state before
```

Property-based testing generates many inputs for such rules.

The property must still be written correctly.

Automation cannot repair a vague or false oracle.

## 28. A Reference Model Simplifies Stateful Testing

A model is a simpler representation of the expected behavior.

Suppose the production structure is a dynamic buffer.

For a bounded test of at most `256` elements, the model can be:

```c
typedef struct {
    int values[256];
    size_t size;
} BufferModel;
```

Model push is simple:

```c
static bool model_push(BufferModel *model, int value) {
    if (model->size == 256) {
        return false;
    }

    model->values[model->size] = value;
    ++model->size;
    return true;
}
```

After every generated operation, compare:

```txt
production size == model size
production elements == model elements
production invariant holds
```

The model is intentionally less sophisticated.

It has:

- fixed capacity
- no reallocation
- no allocator seam
- no amortized growth

That simplicity makes it less likely to repeat the production bug.

### Model-Based Flow

```txt
generated operation
       |
       +------------------+
       |                  |
       v                  v
production buffer     simple model
       |                  |
       +--------+---------+
                |
                v
        compare observable state
```

For later structures:

- a stack can be modeled by an array
- a queue can be modeled by a simple shifting array
- a set can be modeled by a sorted unique array
- a priority queue can be modeled by scanning an unsorted array for the minimum

The model may be slower.

For testing small generated cases, clarity is more important than performance.

## 29. Randomized Tests Must Be Reproducible

Random operation sequences can explore combinations I did not think to write manually.

But:

> A failure that cannot be reproduced is difficult to debug and preserve.

The test must record:

- the seed
- the operation sequence, or enough information to regenerate it
- the first failing step

A small deterministic generator can be:

```c
static uint32_t next_random(uint32_t *state) {
    uint32_t value = *state;

    value ^= value << 13;
    value ^= value >> 17;
    value ^= value << 5;

    *state = value;
    return value;
}
```

The seed must be nonzero for this generator:

```c
uint32_t seed = UINT32_C(0xA341316C);
uint32_t state = seed;
```

The seed should be printed before or upon failure:

```c
fprintf(
    stderr,
    "random seed: 0x%08" PRIX32 "\n",
    seed
);
```

### Generated Test Sketch

The following function can be added to the complete laboratory:

```c
typedef struct {
    int values[256];
    size_t size;
} BufferModel;

static uint32_t next_random(uint32_t *state) {
    uint32_t value = *state;

    value ^= value << 13;
    value ^= value >> 17;
    value ^= value << 5;

    *state = value;
    return value;
}

static void test_random_push_get_sequence(
    TestContext *context
) {
    const uint32_t seed = UINT32_C(0xA341316C);
    uint32_t random_state = seed;
    size_t failures_before = context->failures;

    BufferModel model = {
        .values = {0},
        .size = 0,
    };

    IntBuffer buffer;
    bool initialized = int_buffer_init(
        &buffer,
        system_allocator()
    );

    REQUIRE_TRUE(context, initialized, cleanup);

    for (size_t step = 0; step < 1000; ++step) {
        uint32_t choice = next_random(&random_state);

        if ((choice % 3U) != 0U && model.size < 256) {
            int value = (int)(next_random(&random_state) % 10000U);

            REQUIRE_TRUE(
                context,
                int_buffer_push(&buffer, value),
                cleanup
            );

            model.values[model.size] = value;
            ++model.size;
        } else {
            size_t index;

            if (model.size == 0) {
                index = 0;
            } else {
                index = (size_t)(
                    next_random(&random_state)
                    % (uint32_t)(model.size + 1)
                );
            }

            int output = -1;
            bool found = int_buffer_get(
                &buffer,
                index,
                &output
            );

            if (index < model.size) {
                EXPECT_TRUE(context, found);
                EXPECT_INT(
                    context,
                    output,
                    model.values[index]
                );
            } else {
                EXPECT_FALSE(context, found);
                EXPECT_INT(context, output, -1);
            }
        }

        EXPECT_SIZE(context, buffer.size, model.size);
        EXPECT_TRUE(context, int_buffer_is_valid(&buffer));

        for (size_t i = 0; i < model.size; ++i) {
            EXPECT_INT(
                context,
                buffer.data[i],
                model.values[i]
            );
        }
    }

cleanup:
    if (context->failures != failures_before) {
        fprintf(
            stderr,
            "reproduce with seed 0x%08" PRIX32 "\n",
            seed
        );
    }

    if (initialized) {
        int_buffer_destroy(&buffer);
    }
}
```

The modulo operations introduce a small distribution bias.

That is acceptable for this exploratory test because the goal is not cryptographic or statistical uniformity.

The goal is a deterministic variety of operations.

Important boundaries should still have explicit deterministic tests.

Random generation complements designed cases.

It does not replace them.

### Shrinking

Property-testing frameworks often shrink a failing generated input.

Shrinking tries to remove irrelevant operations until a smaller failing sequence remains.

For example:

```txt
push 4
reserve 100
get 0
push 9
push 12
clone
destroy source
read clone
```

might shrink to:

```txt
push 4
clone
destroy source
read clone
```

The smaller sequence exposes the essential ownership bug.

Writing a general shrinker is beyond this basic harness, but the idea matters when adopting a property-testing library.

## 30. Sanitizers Observe What Value Assertions Miss

Consider:

```c
buffer.data[buffer.capacity] = 10;
```

The write is one element beyond the allocation.

The program may not crash immediately.

A later value test may even pass.

AddressSanitizer instruments memory accesses and can report the invalid write at its origin.

### AddressSanitizer

Compile:

```sh
cc -std=c17 \
   -Wall -Wextra -Wpedantic -Wconversion -Werror \
   -O1 -g \
   -fsanitize=address \
   -fno-omit-frame-pointer \
   testing_lab.c \
   -o testing_lab_asan
```

Run:

```sh
./testing_lab_asan
```

It can detect many instances of:

- heap buffer overflow
- stack buffer overflow
- global buffer overflow
- use after free
- double free
- invalid free
- some memory leaks, depending on platform support

### UndefinedBehaviorSanitizer

Compile both sanitizers:

```sh
cc -std=c17 \
   -Wall -Wextra -Wpedantic -Wconversion -Werror \
   -O1 -g \
   -fsanitize=address,undefined \
   -fno-omit-frame-pointer \
   testing_lab.c \
   -o testing_lab_san
```

UndefinedBehaviorSanitizer can detect executed cases of:

- signed integer overflow
- invalid shifts
- misaligned access
- some null accesses
- several other undefined operations

Sanitizers do not inspect paths that tests never execute.

An untested bug remains invisible.

### Valgrind Memcheck

On a supported Linux environment, compile an ordinary debug binary:

```sh
cc -std=c17 \
   -Wall -Wextra -Wpedantic -Wconversion -Werror \
   -O0 -g3 \
   testing_lab.c \
   -o testing_lab_debug
```

Then run:

```sh
valgrind \
    --leak-check=full \
    --show-leak-kinds=all \
    --track-origins=yes \
    --error-exitcode=1 \
    ./testing_lab_debug
```

`--error-exitcode=1` converts detected memory errors into an automation failure.

Do not run an AddressSanitizer-instrumented binary under Valgrind.

They are separate instrumentation systems and should normally run in separate jobs.

## 31. Compile in More Than One Mode

A debug build and an optimized build exercise different conditions.

### Debug Build

```sh
cc -std=c17 \
   -Wall -Wextra -Wpedantic -Wconversion -Werror \
   -O0 -g3 \
   testing_lab.c \
   -o testing_lab_debug
```

This favors:

- readable debugging
- direct correspondence with source
- local variable visibility

### Sanitizer Build

```sh
cc -std=c17 \
   -Wall -Wextra -Wpedantic -Wconversion -Werror \
   -O1 -g \
   -fsanitize=address,undefined \
   -fno-omit-frame-pointer \
   testing_lab.c \
   -o testing_lab_san
```

This favors dynamic error detection.

### Release-Like Build

```sh
cc -std=c17 \
   -Wall -Wextra -Wpedantic -Wconversion -Werror \
   -O2 -DNDEBUG \
   testing_lab.c \
   -o testing_lab_release
```

This checks that:

- tests do not depend on standard `assert` side effects
- optimization does not expose undefined behavior
- release configuration still passes the explicit harness

Undefined behavior frequently appears to “work” under `-O0` and fail under optimization.

Both outcomes come from invalid source.

Testing an optimized build is therefore part of C testing, not merely performance work.

## 32. A Makefile Makes the Workflow Repeatable

Typing long compiler commands manually invites drift.

A Makefile records the intended build modes.

```make
CC ?= cc

COMMON_FLAGS := \
	-std=c17 \
	-Wall \
	-Wextra \
	-Wpedantic \
	-Wconversion \
	-Werror

BUILD_DIR := build
SOURCE := testing_lab.c

DEBUG_BIN := $(BUILD_DIR)/testing_lab_debug
SAN_BIN := $(BUILD_DIR)/testing_lab_san
RELEASE_BIN := $(BUILD_DIR)/testing_lab_release

.PHONY: all test sanitize release valgrind clean

all: test

$(BUILD_DIR):
	mkdir -p $(BUILD_DIR)

$(DEBUG_BIN): $(SOURCE) | $(BUILD_DIR)
	$(CC) $(COMMON_FLAGS) -O0 -g3 $< -o $@

$(SAN_BIN): $(SOURCE) | $(BUILD_DIR)
	$(CC) $(COMMON_FLAGS) \
		-O1 -g \
		-fsanitize=address,undefined \
		-fno-omit-frame-pointer \
		$< -o $@

$(RELEASE_BIN): $(SOURCE) | $(BUILD_DIR)
	$(CC) $(COMMON_FLAGS) -O2 -DNDEBUG $< -o $@

test: $(DEBUG_BIN)
	./$(DEBUG_BIN)

sanitize: $(SAN_BIN)
	./$(SAN_BIN)

release: $(RELEASE_BIN)
	./$(RELEASE_BIN)

valgrind: $(DEBUG_BIN)
	valgrind \
		--leak-check=full \
		--show-leak-kinds=all \
		--track-origins=yes \
		--error-exitcode=1 \
		./$(DEBUG_BIN)

clean:
	rm -rf $(BUILD_DIR)
```

Recipe lines must begin with a tab.

Run:

```sh
make test
make sanitize
make release
make valgrind
```

Generated binaries belong under `build/`.

They should not be committed.

A matching `.gitignore` entry is:

```gitignore
build/
```

## 33. Debug the First Broken Transition

When a long test fails, the final symptom may be far from the cause.

A disciplined debugging process is:

1. reproduce with one command
2. identify the first failed expectation or sanitizer report
3. reduce to the smallest failing test
4. find the first state transition where actual and expected diverge
5. inspect ownership and invariants at that transition
6. fix the cause
7. retain the reduced case as a regression test
8. rerun the entire suite under all relevant build modes

### Compile for GDB

```sh
cc -std=c17 \
   -Wall -Wextra -Wpedantic \
   -O0 -g3 \
   testing_lab.c \
   -o testing_lab_debug
```

Start:

```sh
gdb ./testing_lab_debug
```

Useful commands include:

```gdb
break test_push_failure_preserves_state
run
next
step
print buffer
print tracker
print *buffer.data
backtrace
continue
```

If the program crashes:

```gdb
run
backtrace
frame 0
info locals
```

The backtrace shows how execution reached the failure.

### Watch an Invariant Field

After reaching a scope where `buffer` exists:

```gdb
watch buffer.size
continue
```

GDB stops when the field changes.

This can reveal an unexpected write.

Debugger inspection is evidence about one execution.

After finding the cause, encode the scenario as a deterministic test so the bug cannot silently return.

## 34. Coverage Measures Execution, Not Correctness

Line coverage asks:

> Which instrumented source lines executed?

Branch coverage asks:

> Which decision outcomes executed?

Coverage can reveal untested code.

For example, if this failure branch never executes:

```c
if (resized == NULL) {
    return false;
}
```

then the suite has not tested allocation failure.

That is useful information.

But one executed line can still contain:

- a wrong formula
- a weak oracle
- an untested value boundary
- an ownership error not observed by assertions

One hundred percent line coverage does not imply correctness.

### GCC Coverage Example

Compile:

```sh
cc -std=c17 \
   -Wall -Wextra -Wpedantic \
   -O0 -g --coverage \
   -c testing_lab.c \
   -o testing_lab.o

cc --coverage \
   testing_lab.o \
   -o testing_lab_coverage
```

Run:

```sh
./testing_lab_coverage
gcov testing_lab.c
```

The generated report marks:

- executed lines
- execution counts
- unexecuted lines

Coverage should guide questions:

- Why is this line untested?
- Is this branch reachable?
- Which state triggers it?
- Does this error path preserve the invariant?

Chasing a percentage without answering those questions produces shallow tests.

## 35. Mutation Testing Challenges the Oracle

Coverage asks whether code ran.

Mutation testing asks whether tests notice when code is made wrong.

Imagine changing:

```c
++buffer->size;
```

to:

```c
buffer->size += 2;
```

A size test should fail.

Now imagine changing:

```c
if (index >= buffer->size) {
```

to:

```c
if (index > buffer->size) {
```

A boundary test at:

```txt
index == size
```

should fail under ASan or an explicit expectation.

Other useful manual mutations are:

- remove the null check
- update capacity before checking allocation success
- replace deep clone with structure assignment
- omit destruction of old destination storage
- increment size before writing the new value
- change an overflow guard

If all tests still pass, the oracle or case selection is too weak.

Automated mutation tools can generate many such changes.

Even manual mutation is a powerful review method for a small C project.

## 36. Organize Tests Around Behavior

Test names should describe behavior:

```txt
push failure preserves state
clone is independent
destroy releases and resets
get boundaries preserve output
```

These are weaker names:

```txt
test 1
push test
buffer stuff
edge case
```

A useful project layout later may be:

```txt
code/
    int_buffer/
        int_buffer.h
        int_buffer.c
        Makefile
        tests/
            test_main.c
            test_init.c
            test_push.c
            test_clone.c
            test_failures.c
            test_support.h
            test_support.c
```

The exact split should follow project size.

For a small chapter laboratory, one translation unit is easier to read and compile.

For a growing implementation:

- production declarations belong in headers
- production definitions belong in source files
- test support should not leak into the production API without a reason
- tests should compile against the same public header as real callers

The build should not copy production functions into test files.

Otherwise the test may exercise a duplicate instead of the shipped implementation.

## 37. Common Broken Testing Patterns

### Testing Only the Happy Path

```c
int_buffer_push(&buffer, 10);
EXPECT_INT(context, buffer.data[0], 10);
```

Missing observations include:

- return value
- size
- capacity relationship
- invariant
- cleanup
- failure behavior

### Ignoring Return Values

```c
int_buffer_push(&buffer, 10);
EXPECT_INT(context, buffer.data[0], 10);
```

If push failed, reading `data[0]` may itself be invalid.

The test should establish the prerequisite before dependent observations.

### Returning Before Cleanup

```c
if (!int_buffer_push(&buffer, 10)) {
    return;
}
```

If the buffer already owns storage, this leaks during the test.

Use one cleanup path or a fixture abstraction.

### Repeating the Implementation

If the production capacity formula and expected formula are copied from the same source, the test can repeat the same bug.

Prefer contract-level properties unless exact growth is part of the public promise.

### Over-Specifying Representation

```c
EXPECT_SIZE(context, buffer.capacity, 8);
```

This is wrong when the contract promises only:

```txt
capacity >= requested capacity
```

### Under-Specifying Failure

```c
EXPECT_FALSE(context, pushed);
```

This checks the status but not whether the old state was corrupted.

### One Giant Test

A thousand-operation scenario with one final assertion is difficult to diagnose.

Check intermediate transitions and keep focused deterministic cases.

### Shared Mutable Fixtures

Tests that depend on execution order are difficult to isolate and parallelize.

### Hidden Random Seed

A random failure without a seed may disappear on the next run.

### Disabled Assertions Remove Actions

```c
assert(int_buffer_push(&buffer, 10));
```

Under `NDEBUG`, the push is removed.

### Tests That Leak Because “The Process Exits Anyway”

The operating system may reclaim memory at exit.

The leak still indicates broken ownership and pollutes leak-checker output.

Test cleanup should meet the same standard as production cleanup.

### Comparing Owning Structures With `memcmp`

Pointer values, capacity slack, and padding are not logical equality.

Compare the structure's promised observable content.

## 38. Common Misconceptions

### “Passing Tests Prove There Are No Bugs”

No.

They show that executed observations matched their oracles.

### “More Tests Always Mean Better Tests”

No.

Case diversity, oracle strength, and risk coverage matter more than raw count.

### “Every Function Needs Exactly One Unit Test”

No.

A simple accessor may need several boundary cases.

Several internal helpers may be sufficiently covered through one public behavior.

### “One Hundred Percent Coverage Means Correct”

No.

Executed wrong logic is still wrong.

### “Sanitizers Replace Assertions”

No.

Sanitizers detect classes of runtime violations.

They do not know the intended logical result.

### “Assertions Replace Error Handling”

No.

Expected runtime failures need defined return behavior.

Assertions express programmer assumptions.

### “Random Tests Find Everything”

No.

They may repeatedly miss a narrow boundary unless generation targets it.

### “Mocks Make a Test a Unit Test”

No.

Unnecessary mocks can test an invented interaction rather than real behavior.

Use a seam when control or observation is otherwise impossible, as with deterministic allocation failure.

### “Private Fields Should Never Be Tested”

Not absolutely.

Most tests should prefer public behavior, but difficult internal invariants may justify targeted white-box checks.

### “Tests Should Duplicate the Requirement in Code”

They should encode the requirement, but their oracle should not blindly duplicate the production algorithm.

### “A Crash Is a Good Enough Failure Message”

No.

A test name, source location, state snapshot, and sanitizer backtrace reduce debugging time dramatically.

### “If a Bug Was Fixed, the Test Is No Longer Needed”

The test becomes the regression barrier that preserves the fix.

## 39. A Repeatable Testing Workflow

For each data-structure operation, I will:

1. Write its preconditions.
2. Write its success result and state transition.
3. Write its failure result and preservation guarantees.
4. List the invariant before and after.
5. Identify empty, one-element, full, and boundary states.
6. Choose an oracle independent of the implementation where possible.
7. Write one focused normal-path test.
8. Write boundary tests.
9. Write failure-path tests.
10. Check outputs that must remain unchanged.
11. Check old values that must be preserved.
12. Check ownership and cleanup.
13. Run strong compiler warnings.
14. Run debug, sanitizer, and release-like builds.
15. Add a model or property test when sequences become complex.
16. Record seeds for generated cases.
17. Reduce every discovered bug to a regression test.
18. Review uncovered and unkillable branches rather than chasing a score.

The workflow starts with the contract.

The test code is downstream of that reasoning.

## 40. Practice Questions

For every implementation exercise:

1. state the contract
2. identify the oracle
3. describe the fixture
4. identify cleanup
5. include boundary and failure behavior
6. explain which tool adds evidence beyond value assertions

### Question 1: Specify Before Testing

Write a complete contract for:

```c
bool int_buffer_pop(
    IntBuffer *buffer,
    int *value_out
);
```

Decide:

- behavior for null arguments
- behavior for an empty buffer
- whether output changes on failure
- whether capacity shrinks
- which invariant must remain true

Then derive the minimum distinct tests from that contract.

### Question 2: Repair a Dangerous Macro

Explain every problem:

```c
#define EXPECT_EQ(a, b) \
    if (a != b) printf("%d != %d\n", a, b)
```

Address:

- repeated evaluation
- missing type discipline
- interaction with `if` and `else`
- missing file and line
- missing failure count
- incorrect format specifiers

Write safe `int` and `size_t` versions.

### Question 3: Cleanup-Safe Prerequisites

This test leaks on failure:

```c
static void test_something(TestContext *context) {
    IntBuffer buffer;
    int_buffer_init(&buffer, system_allocator());
    int_buffer_push(&buffer, 10);

    if (!condition()) {
        return;
    }

    int_buffer_destroy(&buffer);
}
```

Rewrite it with:

- checked initialization
- a fatal prerequisite
- one cleanup section
- no use of an invalid fixture

### Question 4: Empty State Matrix

Create a table of expected behavior for every public `IntBuffer` operation when:

- the buffer pointer is null
- the buffer is uninitialized
- the buffer is initialized and empty
- the buffer is empty with reserved capacity
- an output pointer is null

Separate valid contract cases from caller precondition violations.

### Question 5: Boundary Transition Tests

Assume capacities double.

Design a push sequence that tests:

- zero to one
- one to two
- two to three
- three to four
- four to five

For each step, state which assertions depend only on the public contract and which depend on the chosen growth policy.

### Question 6: Failure-Atomic Reserve

Design a deterministic test proving that failed:

```c
int_buffer_reserve(&buffer, 1000)
```

preserves:

- pointer
- size
- capacity
- values
- allocator ownership count
- later usability

Explain why checking only `false` is weak.

### Question 7: Track Exact Allocations

The chapter tracker counts live blocks but does not maintain a registry of pointer identities.

Design a stronger test allocator that records up to `128` live pointers.

It must detect:

- freeing an unknown pointer
- freeing the same pointer twice
- leaked pointers
- relocation by `realloc`

Explain how to update the registry when `realloc` succeeds, fails, or returns the same address.

### Question 8: Multi-Allocation Clone

Imagine a structure owns:

```c
char *name;
int *values;
```

Cloning requires two allocations.

Write failure tests for:

- first allocation fails
- first succeeds and second fails
- both succeed

Prove destination preservation and absence of temporary leaks in every case.

### Question 9: Opaque Representation

Change `IntBuffer` so its fields are hidden in `int_buffer.c`.

Design public queries sufficient to test:

- size
- element values
- validity of indices
- clone independence

Which white-box tests would be lost?

Should a debug-only invariant function be exposed?

Defend the decision.

### Question 10: Stack Model

Build a stack implementation and model it with:

```c
int model[256];
size_t model_size;
```

Generate deterministic sequences of:

- push
- pop
- peek

Compare production and model after every operation.

Record the seed and failing step.

### Question 11: Shrink a Failure

Given this failing sequence:

```txt
push 3
push 8
reserve 40
clone
push 12 to clone
destroy source
get clone index 0
```

Develop a manual shrinking procedure.

Find the smallest subsequence that still demonstrates a shallow-copy bug.

### Question 12: Sanitizer Diagnosis

Insert this bug into push:

```c
buffer->data[buffer->capacity] = value;
```

Predict:

- which boundary first makes it invalid
- whether an ordinary value test must detect it
- what AddressSanitizer should report

Then fix the code and retain a regression test.

### Question 13: Release-Only Failure

A test passes under:

```txt
-O0
```

and fails under:

```txt
-O2 -DNDEBUG
```

List plausible causes involving:

- undefined behavior
- uninitialized values
- assertion side effects
- strict aliasing
- lifetime

Describe a debugging sequence.

### Question 14: Coverage Review

Suppose the suite reports ninety-eight percent line coverage.

The only uncovered lines handle capacity-overflow rejection.

Explain why those lines matter despite the high percentage.

Design a test that reaches the guard without attempting an enormous allocation.

Would the current public API make that easy?

### Question 15: Manual Mutation Review

Apply five mutations to `int_buffer_get` and `int_buffer_push`.

For each mutation:

- predict which test should fail
- run the suite
- identify any surviving mutation
- strengthen the suite without over-specifying implementation

### Question 16: Table-Driven Get Tests

Write a complete table-driven test for a three-element buffer.

Include:

- indices `0`, `1`, and `2`
- index `3`
- `SIZE_MAX`
- null output
- sentinel preservation on every failure

Make diagnostics identify which row failed.

### Question 17: Reserve Properties

State properties for:

```c
int_buffer_reserve(buffer, requested)
```

Cover:

- request below capacity
- request equal to capacity
- request above capacity
- overflow
- allocation failure

Distinguish logical size, physical capacity, pointer identity, and value preservation.

### Question 18: Test a Ring Buffer

For a fixed-capacity circular queue, identify states where:

```txt
head > tail
head == tail
head < tail
```

Explain why `head == tail` may mean empty or full unless another field or rule disambiguates it.

Design transition tests for wraparound.

### Question 19: Regression From a Bug Report

A user reports:

> After cloning a nonempty buffer into an already populated destination, one allocation leaks.

Write the smallest deterministic regression test.

State:

- initial owners
- allocator observations
- operation
- logical assertions
- cleanup assertions

### Question 20: Full Linked-List Test Plan

Without implementing the list yet, write a complete test plan for:

- initialize
- push front
- push back
- find
- remove first match
- pop front
- clone
- destroy

Include:

- empty and one-node states
- head and tail removal
- duplicate values
- allocation failure at each allocating operation
- deep-copy independence
- cycle-related preconditions
- ownership and leak checks
- a simple reference model

## 41. Final Check

Before moving on, I should be able to explain:

- why a test is an executable observation rather than a proof of all behavior
- why contracts and invariants come before cases
- what makes an oracle strong but not over-specific
- why state transitions matter more than isolated values
- why empty, full, and failure states need deliberate tests
- why cleanup belongs inside test correctness
- why fatal prerequisites should still reach cleanup
- why assertion macros must evaluate arguments once
- why test order must not matter
- how an allocator seam makes failure deterministic
- how to prove failure atomicity
- why deep-copy equality is weaker than copy independence
- how a reference model differs from the production structure
- why randomized tests need reproducible seeds
- what sanitizers observe that logical assertions do not
- why debug, sanitizer, and release-like builds all matter
- why coverage measures execution rather than correctness
- how mutation testing challenges oracle strength
- why every fixed bug should leave behind a regression test

If I cannot state what a test observes, where its expected result comes from, and what remains untested, I do not yet understand the evidence that test provides.
