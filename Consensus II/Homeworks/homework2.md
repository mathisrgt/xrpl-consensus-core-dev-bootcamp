# Homework 2: Negative UNL Engineering

[← Back to Consensus II](./)

***

### Objective

This homework provides hands-on experience with **XRPL's Negative UNL (Universal Node List) mechanism**. You will create test scenarios that disable and re-enable validators, observing how the network maintains consensus despite validator failures.

**Format**: Written report (PDF or Markdown) with test code, execution output, and analysis.

***

### Background

The Negative UNL mechanism allows the network to temporarily disable poorly performing validators without manual intervention:

**Key Concepts:**
- **Flag Ledgers**: Updates occur every 256 ledgers
- **ToDisable**: Validators marked for disabling via `ttUNL_MODIFY` transactions
- **ToReEnable**: Disabled validators marked for re-enabling when recovered
- **Scoring**: Validators scored based on validation participation using `validationFRESHNESS`
- **Thresholds**: Low/high watermarks determine disable/re-enable actions

**Transaction Types:**
- **ttUNL_MODIFY**: Special transaction type for UNL modifications
- Only one ToDisable per flag ledger
- Only one ToReEnable per flag ledger
- Transactions processed automatically at flag ledgers

***

### Task

Create a comprehensive test function that simulates the complete validator lifecycle: disable, operation while disabled, and re-enable.

#### Requirements

1. **Test File Setup**
   * Navigate to `rippled/src/test/app/`
   * Open `LedgerMaster_test.cpp` or create new test file
   * Locate the `testNegativeUNL()` function for reference (~line 100+)
   * Understand the test helper functions

2. **Phase 1: Initial Setup** (15 minutes)
   * Create test function `testValidatorLifecycle()`
   * Create 5 validator public keys
   * Initialize genesis ledger with `featureNegativeUNL` enabled
   * Create tracking structure: `hash_map<PublicKey, std::uint32_t> nUnlLedgerSeq`
   * Add logging to track progress

3. **Phase 2: Flag Ledger Progression** (20 minutes)
   * Build ledgers until first flag ledger (sequence 256)
   * Verify `l->isFlagLedger()` returns true
   * Call `l->updateNegativeUNL()` on flag ledger
   * Verify initial state: empty nUNL, no ToDisable, no ToReEnable
   * Use `negUnlSizeTest()` helper for verification

4. **Phase 3: Validator Disabling** (25 minutes)
   * Create `ttUNL_MODIFY` transaction for validator 0 (ToDisable)
   * Create second disable transaction for validator 1 (should fail)
   * Create re-enable transaction for validator 0 (should fail - not in nUNL yet)
   * Apply transactions using `OpenView`
   * Verify only first disable succeeds
   * Verify ToDisable field is set

5. **Phase 4: Process Disable** (20 minutes)
   * Progress 256 ledgers to next flag ledger
   * Verify ToDisable persists during progression
   * Call `updateNegativeUNL()` on new flag ledger
   * Verify validator 0 is now in nUNL
   * Verify ToDisable field is cleared
   * Record validator 0's disable ledger sequence

6. **Phase 5: Complex Scenarios** (20 minutes)
   * Test disabling already-disabled validator (should fail)
   * Test disabling new validator 2 (should succeed)
   * Test re-enabling validator not in nUNL (should fail)
   * Test re-enabling same validator as ToDisable (should fail)
   * Test re-enabling validator 0 from nUNL (should succeed)
   * Verify final state has both ToDisable and ToReEnable set

7. **Phase 6: Complete Lifecycle** (15 minutes)
   * Progress to next flag ledger
   * Process both ToDisable and ToReEnable
   * Verify validator 2 added to nUNL
   * Verify validator 0 removed from nUNL
   * Verify tracking map is accurate

***

### Implementation Guide

**Test Function Skeleton:**

```cpp
void testValidatorLifecycle()
{
    testcase("Validator Lifecycle - Disable and Re-enable");

    std::cout << "=== Starting Validator Lifecycle Test ===" << std::endl;

    // Setup
    jtx::Env env(*this, jtx::supported_amendments() | featureNegativeUNL);
    std::vector<PublicKey> publicKeys = createPublicKeys(5);

    auto l = std::make_shared<Ledger>(
        create_genesis,
        env.app().config(),
        std::vector<uint256>{},
        env.app().getNodeFamily());

    hash_map<PublicKey, std::uint32_t> nUnlLedgerSeq;

    // Phase 1: Build to first flag ledger
    std::cout << "\n=== Building to First Flag Ledger ===" << std::endl;
    // ... your code here

    // Phase 2: Test disable transactions
    std::cout << "\n=== Testing Validator Disable ===" << std::endl;
    // ... your code here

    // Phase 3: Process disable
    std::cout << "\n=== Processing Disable ===" << std::endl;
    // ... your code here

    // Phase 4: Test complex scenarios
    std::cout << "\n=== Testing Complex Scenarios ===" << std::endl;
    // ... your code here

    // Phase 5: Complete lifecycle
    std::cout << "\n=== Completing Lifecycle ===" << std::endl;
    // ... your code here

    std::cout << "\n=== Test Complete ===" << std::endl;
}
```

**Helper Functions Available:**
```cpp
// Create disable/enable transaction
auto tx = createTx(bool isToDisable, LedgerIndex seq, PublicKey const& key);

// Apply transaction and test result
bool result = applyAndTestResult(Env& env, OpenView& view, STTx const& tx, bool shouldSucceed);

// Test nUNL state
bool ok = negUnlSizeTest(Ledger const& l, size_t expectedSize, bool hasToDisable, bool hasToReEnable);

// Verify tracking map
bool verified = VerifyPubKeyAndSeq(Ledger const& l, hash_map<PublicKey, uint32_t> const& map);
```

***

### Deliverable

A written report containing:

* **Complete Test Code**:
  * Full `testValidatorLifecycle()` function implementation
  * Comments explaining each phase
  * Proper use of helper functions
  * BEAST_EXPECT assertions at key points

* **Test Execution Output**:
  * Console output from running your test
  * Show all 6 phases completing
  * Show success/failure of each transaction
  * Show nUNL state changes

* **Analysis Table**:

  | Ledger Sequence | Event | nUNL Size | ToDisable | ToReEnable |
  |-----------------|-------|-----------|-----------|------------|
  | 256 (1st flag) | Initial | 0 | None | None |
  | 256 | Disable TX | 0 | Validator 0 | None |
  | 512 (2nd flag) | Process Disable | 1 | None | None |
  | 512 | Complex TXs | 1 | Validator 2 | Validator 0 |
  | 768 (3rd flag) | Complete Lifecycle | 1 | None | None |

* **Questions Answered**:
  1. Why can only one ToDisable transaction be processed per flag ledger?
  2. Why can't a validator be disabled before reaching a flag ledger?
  3. What happens to the ToDisable field after it's processed?
  4. Can a validator be in both ToDisable and ToReEnable simultaneously?
  5. How does the 256-ledger interval affect network responsiveness to validator failures?
  6. What would happen if you tried to disable all validators?

***

### Advanced Challenges (Optional)

1. **Scoring System**: Implement validator scoring based on `validationFRESHNESS`
2. **Automatic Disable**: Trigger automatic disable when score falls below threshold
3. **Multiple Cycles**: Test disabling and re-enabling the same validator multiple times
4. **Edge Cases**: Test with only 2 validators, or with all but one disabled

***

### Tips

* Use `std::cout` liberally to track progress through phases
* The `BEAST_EXPECT()` macro is your test assertion
* Flag ledgers are at sequences 256, 512, 768, 1024, etc.
* `updateNegativeUNL()` must be called on each flag ledger
* OpenView allows transaction simulation before applying to ledger
* Read the existing `testNegativeUNL()` function for patterns
* Build and run: `cmake --build . --target rippled --parallel 10 && ./rippled --unittest=LedgerMaster`

***

### Learning Goals

By completing this homework, you should be able to:

* Write consensus tests using the jtx framework
* Understand the Negative UNL transaction lifecycle
* Observe automatic validator management mechanisms
* Explain flag ledger processing
* Predict nUNL state changes based on transactions
* Debug consensus test failures
* Verify complex multi-phase consensus scenarios

***

### Reference Materials

* **Negative UNL Implementation**: `rippled/src/xrpld/app/misc/NegativeUNLVote.cpp`
* **Test Framework**: `rippled/src/test/jtx/`
* **Ledger Interface**: `rippled/src/xrpld/ledger/ReadView.h`
