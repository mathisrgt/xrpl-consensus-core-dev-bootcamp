# Amendments

[← Back to Consensus and Ledger Architecture](https://docs.xrpl-commons.org/core-dev-bootcamp/module08)

***

## Introduction

Amendments are XRPL's democratic governance system for protocol evolution. They enable the network to evolve and improve while maintaining consensus and decentralization—allowing protocol changes to be coordinated across thousands of nodes without a central authority.

### What Are Amendments?

**Definition:** An amendment is a proposed change to the XRPL's core rules that affects transaction processing logic, consensus mechanisms, ledger structure, or network behavior.

**Key Characteristics:**

**Backward Compatibility:**
- Must not break existing functionality
- Graceful activation without network disruption
- Code exists dormant until activated

**Network-Wide Impact:**
- Affects all participants equally
- All nodes must comply simultaneously
- Synchronized activation across network

**Irreversible:**
- Once activated, cannot be undone
- Permanent change to protocol
- Requires careful consideration before activation

**Democratic:**
- Requires supermajority approval (80%+)
- Sustained support over 2+ weeks
- Any significant minority can veto (21%+)

### Why Amendments Matter

**The Challenge:**
- Blockchain networks must evolve to remain competitive
- All participants must agree on the same rules
- Changes must be coordinated across thousands of nodes
- No central authority can force updates

**The Solution:**
- Democratic voting system for protocol changes
- Ensures network-wide consensus before activation
- Maintains decentralization while enabling evolution
- Transparent and auditable process

***

## Amendment Lifecycle

The journey of an amendment from idea to activation involves six distinct phases:

### Phase 1: Community Proposal

**The Democratic Process:**

**Anyone Can Propose:**
- Developers, businesses, community members
- Open submission process
- No gatekeepers or central approval

**Public Discussion:**
- Ideas debated openly in forums and GitHub
- Technical analysis and impact assessment
- Community feedback and refinement

**Consensus Building:**
- Gathering community support
- Building coalition for approval
- Addressing concerns and objections

**Refinement:**
- Proposals improved through feedback
- Technical specifications refined
- Edge cases identified and addressed

**Key Considerations:**

- **Need Identification:** What problem does this solve?
- **Impact Assessment:** Who benefits? Who might be affected?
- **Technical Feasibility:** Can this be implemented safely?
- **Community Support:** Is there genuine demand?

### Phase 2: Development & Testing

**Development Process:**

**Technical Specification:**
- Detailed requirements written
- Formal specification document
- Clear success criteria

**Code Implementation:**
- Developers write the actual changes
- Pull request submitted to rippled repository
- Code review by maintainers and community

**Rigorous Testing:**
- Unit tests for individual components
- Integration tests for system-wide behavior
- Security audits for potential vulnerabilities
- Performance analysis for network impact

**Quality Assurance:**
- Multiple rounds of testing
- Peer review by other developers
- Community testing on test networks
- Stress testing under various conditions

### Phase 3: Network Deployment

**Code Distribution:**

**Pull Request Merge:**
- Code approved and merged to main repository
- Included in next rippled release
- Version number assigned

**Build Integration:**
- Amendment code included in rippled software
- Compilation and packaging
- Distribution via official channels

**Node Updates:**
- Validators update their software
- Amendment code present but inactive
- Dormant state—no immediate effect

**The Waiting Period:**
- Network behavior remains unchanged
- Nodes confirm they have the update
- System prepares for voting
- Validators configure their preferences

### Phase 4: Voting Period

**Voting Participants:**

**Validators Only:**
- Only nodes that participate in consensus can vote
- Weighted equally—each validator has one vote
- Public positions—all votes transparent and verifiable

**Voting Expression:**

**Continuous Signaling:**
- Validators constantly express preferences in validations
- Amendment flags indicate support
- Flexible timing—can change vote anytime

**Configuration:**
```
[amendments]
# Support specific amendment
B2A4DB846F0891BF2C76AB2F2ACC8F5B4EC64437135C6E56F3F859DE5FFD5856

[veto_amendments]
# Veto specific amendment
C4483A1896170C66C098DEA5B0E024309C60DC960DE5F01CD7AF986AA3D9AD37
```

**Vote Collection:**
- Votes collected from trusted validations
- Aggregated each consensus round
- Threshold calculated based on trusted validator count

### Phase 5: Threshold Achievement & Activation

**The 80% Threshold:**

**Why 80%?**
- **Strong Consensus:** Ensures broad community support
- **Minority Protection:** Prevents tyranny of simple majority (51%)
- **Network Stability:** Reduces risk of contentious splits
- **Coordination Assurance:** High confidence in network-wide adoption

**Comparison with Other Systems:**
- Simple majority (51%): Too risky for irreversible changes
- Unanimity (100%): Would prevent any progress
- 80%: Sweet spot between progress and stability

**Two-Week Requirement:**

**Purpose of Sustained Support:**
- **Prevents Hasty Decisions:** Allows time for reflection
- **Confirms Stability:** Ensures support isn't temporary
- **Enables Coordination:** Gives network time to prepare
- **Allows Opposition:** Provides opportunity for concerns to emerge

**Dynamic Nature:**
- Continuous monitoring of support levels
- Vote changes tracked in real-time
- Reset mechanism if support drops below 80%
- Clock restarts when threshold lost

**Activation Trigger:**

**Automatic Process:**
- No human intervention required
- 80%+ support for 2+ weeks triggers activation
- Network-wide effect—all nodes comply simultaneously
- Irreversible change—no going back

**Coordination:**
- Synchronized activation at specific ledger
- All nodes switch at same moment
- Consensus ensures smooth transition
- Service continues uninterrupted

### Phase 6: Network Integration

**For Node Operators:**

**Mandatory Compliance:**
- Must use new rules or be excluded from consensus
- Nodes with old rules cannot participate
- Amendment-blocked nodes stop processing

**Software Updates:**
- May need to upgrade rippled versions
- Download and install latest release
- Restart node with new code

**Monitoring Requirements:**
- Track amendment status via RPC
- Monitor for upcoming activations
- Prepare for changes proactively

**Operational Changes:**
- May need to adjust configurations
- Update monitoring and alerting
- Review documentation for changes

**For Network Users:**

**New Capabilities:**
- Access to enhanced features
- Improved transaction types
- Better performance or security

**Behavior Changes:**
- Some operations may work differently
- New validation rules
- Updated fee structures

**Compatibility:**
- Applications may need updates
- APIs might have new fields
- Client libraries require updates

**Improved Experience:**
- Generally benefits from enhancements
- Bug fixes and security improvements
- Better network performance

***

## Amendment Architecture

### AmendmentTable Interface

**Location:** `rippled/src/xrpld/app/misc/AmendmentTable.h`

**Purpose:** Provides methods for managing, voting on, and querying amendment status

**Key Methods:**

**Amendment Management:**
- `find(amendmentID)` - Look up amendment by hash
- `veto(amendmentID)` - Veto a specific amendment
- `unVeto(amendmentID)` - Remove veto
- `enable(amendmentID)` - Enable an amendment

**Status Queries:**
- `isEnabled(amendmentID)` - Check if amendment is active
- `isSupported(amendmentID)` - Check if this node supports it
- `hasUnsupportedEnabled()` - Are there unsupported active amendments?
- `firstUnsupportedExpected()` - When will unsupported amendment activate?

**Voting and Validation:**
- `doVoting(parentCloseTime, amendments, majorityAmendments)` - Determine voting actions
- `doValidation(ledger, enabled)` - Get amendments to include in validation
- `getDesired()` - Get all amendments this node wants enabled

**Reporting:**
- `getJson(majority)` - Get JSON representation of amendment state

### AmendmentTableImpl Implementation

**Location:** `rippled/src/xrpld/app/misc/detail/AmendmentTable.cpp`

**Internal State:**

**Amendment Map:**
```cpp
hash_map<uint256, AmendmentState> amendmentMap_
```

Tracks state of all known amendments

**Voting Tracking:**
```cpp
TrustedVotes previousTrustedVotes_;
std::unique_ptr<AmendmentSet> lastVote_;
```

Maintains validator votes and aggregation

**Database:**
```cpp
DatabaseCon& db_;
```

Persists voting preferences

**Thread Safety:**
```cpp
std::mutex mutex_;
```

Protects all internal state

### AmendmentState Structure

Each amendment tracked with:

```cpp
struct AmendmentState {
    AmendmentVote vote;      // up, down, or obsolete
    bool enabled;            // Currently active?
    bool supported;          // Does this node support it?
    std::string name;        // Human-readable name
};
```

**AmendmentVote Enum:**
- `up` - Vote to enable this amendment
- `down` - Veto this amendment
- `obsolete` - Amendment is obsolete, don't vote

***

## Amendment Registration

### Supported, Obsolete, and Retired Amendments

**Location:** `rippled/include/xrpl/protocol/detail/features.macro`

Amendments registered at compile time using macros:

**Active Amendments:**
```cpp
XRPL_FEATURE(FeatureName, Supported, DefaultVote)
```

**Fixes (Bug Fixes):**
```cpp
XRPL_FIX(FixName, Supported, DefaultVote)
```

**Retired Amendments:**
```cpp
XRPL_RETIRE(RetiredFeature)
```

**Amendment Types:**

**Supported Amendments:**
- Currently relevant features
- Code exists and is maintained
- Can be voted on and activated

**Obsolete Amendments:**
- Marked with `VoteBehavior::Obsolete`
- No longer relevant but must remain supported
- In case they were ever enabled historically
- Don't actively vote on them

**Retired Amendments:**
- Active for 2+ years
- Pre-amendment code removed
- Identifiers deprecated but preserved
- Always considered enabled

### Process for Registering New Amendments

**Steps:**

1. **Add to features.macro:**
   - Use `XRPL_FEATURE` for new features
   - Use `XRPL_FIX` for bug fixes
   - Provide unique identifier and name

2. **Increment Feature Count:**
   - Update `numFeatures` in `Feature.h`
   - Ensures proper array sizing

3. **Implement Feature Logic:**
   - Add conditional code checking `rules.enabled(featureName)`
   - Implement new behavior when enabled
   - Maintain old behavior when disabled

4. **Add Tests:**
   - Unit tests for both enabled/disabled states
   - Integration tests for activation
   - Edge case coverage

5. **Documentation:**
   - Update release notes
   - Document behavior changes
   - Provide migration guidance

***

## Voting Process

### Vote Collection and Aggregation

**TrustedVotes Class:**

**Purpose:** Tracks votes from trusted validators and manages timeouts

**Functionality:**
- Collects amendment votes from validations
- Filters for trusted validators only
- Handles vote expiration
- Aggregates votes per amendment

**Location:** `rippled/src/xrpld/app/misc/detail/AmendmentTable.cpp`

### AmendmentSet Class

**Purpose:** Aggregates votes and determines which amendments pass

**Structure:**
```cpp
class AmendmentSet {
    hash_map<uint256, int> votes_;        // Votes per amendment
    int trustedValidations_;              // Total trusted validators
    int threshold_;                       // Votes needed to pass
};
```

**Threshold Calculation:**

**computeThreshold() Method:**

```cpp
static int computeThreshold(int trustedValidations, Rules const& rules) {
    if (rules.enabled(fixAmendmentMajorityCalc))
        return postFixAmendmentMajorityCalcThreshold(trustedValidations);
    else
        return preFixAmendmentMajorityCalcThreshold(trustedValidations);
}
```

**Pre-Fix Threshold:**
- Typically 80% of trusted validators
- At least 1 vote required
- `(trustedValidations * 8) / 10`

**Post-Fix Threshold:**
- More precise calculation
- Handles rounding better
- Prevents edge cases

**passes() Method:**

Determines if an amendment has enough votes:

```cpp
bool passes(uint256 const& amendment) const {
    auto const& it = votes_.find(amendment);
    if (it == votes_.end())
        return false;
    return it->second >= threshold_;
}
```

### Ledger Integration: Voting Ledgers

**Every 256 Ledgers:**
- Regular voting opportunities
- Predictable schedule
- `FLAG_LEDGER_INTERVAL = 256`

**Embedded in Consensus:**
- Voting part of normal ledger creation
- No disruption to transaction processing
- Automated counting—no human intervention

**Transparent Record:**
- All votes permanently recorded on-ledger
- Audit trail for all changes
- Anyone can verify voting process

**Parallel Processing:**
- Amendments and transactions coexist
- Voting doesn't interfere with operations
- Consistent timing across network

***

## Consensus Integration

### doVoting: Amendment Voting Logic

**Location:** `rippled/src/xrpld/app/misc/detail/AmendmentTable.cpp`

**Called:** Each consensus round on voting ledgers

**Process:**

1. **Update Trusted Votes:**
   - Collect votes from current validations
   - Filter for trusted validators
   - Update vote tracking

2. **Build AmendmentSet:**
   - Aggregate votes per amendment
   - Calculate threshold
   - Determine which amendments pass

3. **For Each Amendment:**

   **If Already Enabled:** Skip

   **Determine Status:**
   - **Validator Majority:** Does it pass vote threshold?
   - **Ledger Majority:** Is it recorded in ledger as having majority?
   - **Time Held:** How long has it had majority?

   **Decide Actions:**
   - **Just Achieved Majority:** Signal `tfGotMajority`
   - **Lost Majority:** Signal `tfLostMajority`
   - **Held 2+ Weeks:** Signal enablement (no flag)
   - **Otherwise:** Log status, no action

4. **Return Actions:**
   - Map of amendment hash to action code
   - `tfGotMajority` (0x00010000)
   - `tfLostMajority` (0x00020000)
   - 0 for enablement

### doValidation: Advertising Support

**Location:** `rippled/src/xrpld/app/misc/detail/AmendmentTable.cpp`

**Purpose:** Determines which amendments to advertise in validation messages

**Process:**

1. Gather all supported amendments
2. Filter for upvoted (not vetoed)
3. Exclude already enabled
4. Sort for deterministic ordering
5. Return vector of amendment hashes

**Included in Validations:**
- Validators broadcast their amendment support
- Other nodes collect these votes
- Aggregated to determine network support

**getDesired() Method:**
- Calls `doValidation` with empty set
- Returns all amendments node wants enabled
- Used for status queries

***

## Ledger Application and Activation

### doValidatedLedger: Synchronizing State

**Location:** `rippled/src/xrpld/app/misc/detail/AmendmentTable.cpp`

**Called:** After each ledger is validated

**Process:**

1. **Enable Ledger Amendments:**
   - Get enabled amendments from ledger
   - Call `enable()` for each
   - Update internal state

2. **Track Majority Amendments:**
   - Get amendments with majority from ledger
   - Update internal tracking
   - Calculate expected activation times

3. **Check for Unsupported:**
   - Identify amendments this node doesn't support
   - Track when they're expected to activate
   - Set `firstUnsupportedExpected_` if needed

4. **Update Last Processed:**
   - Record ledger sequence
   - Prevents duplicate processing

### Change::applyAmendment: Transaction Application

**Location:** `rippled/src/xrpld/app/tx/detail/Change.cpp`

**Purpose:** Applies amendment pseudo-transactions to ledger

**Process:**

1. **Extract Amendment Hash:**
   - Get amendment ID from transaction
   - Validate format

2. **Check if Already Enabled:**
   - If already enabled, return `tefALREADY`
   - Prevent duplicate activation

3. **Handle tfGotMajority Flag:**
   - Add amendment to `sfMajorities` array
   - Record close time when majority achieved
   - Update amendment object in ledger

4. **Handle tfLostMajority Flag:**
   - Remove amendment from `sfMajorities` array
   - Clear majority status
   - Update amendment object

5. **Enable Amendment (No Flags):**
   - Add to `sfAmendments` array
   - Call special activation handlers if needed
   - Call `app.getAmendmentTable().enable(amendment)`
   - **If unsupported:** Log error and block server

6. **Return Success:**
   - Update ledger state
   - Return `tesSUCCESS`

**Special Handling:**

**fixTrustLinesToSelf:**
```cpp
if (amendment == fixTrustLinesToSelf)
    activateTrustLinesToSelfFix(ctx_);
```

Some amendments require special activation logic for data migration or state updates.

***

## Persistence and Database

### Storing Vote Preferences

**persistVote() Method:**

**Location:** `rippled/src/xrpld/app/misc/detail/AmendmentTable.cpp`

**Purpose:** Records vote preference in database

**Process:**
1. Assert vote is not obsolete
2. Get database session
3. Call `voteAmendment()` with details
4. Persist to FeatureVotes table

**voteAmendment() Function:**

**Location:** `rippled/src/xrpld/app/rdb/detail/Wallet.cpp`

**SQL Operation:**
```sql
INSERT INTO FeatureVotes (AmendmentHash, AmendmentName, Vote)
VALUES (?, ?, ?)
```

**Transaction Management:**
- Begin transaction
- Execute insert
- Commit transaction
- Ensures atomicity

### Reading Stored Preferences

**readAmendments() Function:**

**Location:** `rippled/src/xrpld/app/rdb/detail/Wallet.cpp`

**Purpose:** Reads vote preferences from database at startup

**SQL Query:**
```sql
SELECT AmendmentHash, AmendmentName, Vote
FROM (SELECT *, ROW_NUMBER() OVER (PARTITION BY AmendmentHash ORDER BY Timestamp DESC) as rn
      FROM FeatureVotes)
WHERE rn = 1
```

**Process:**
- Use window function to get latest vote per amendment
- Invoke callback for each row
- Caller validates and updates internal state
- Restores preferences across restarts

***

## RPC and Admin Interface

### feature RPC Command

**Location:** `rippled/src/xrpld/rpc/handlers/Feature1.cpp`

**Purpose:** Query and manage amendment status

**Query All Amendments:**
```json
{
  "command": "feature"
}
```

**Query Specific Amendment:**
```json
{
  "command": "feature",
  "feature": "AmendmentNameOrHash"
}
```

**Response Format:**
```json
{
  "features": {
    "AmendmentHash": {
      "name": "AmendmentName",
      "supported": true,
      "enabled": true,
      "vetoed": false,
      "count": 28,
      "threshold": 22,
      "validations": 30
    }
  }
}
```

### Admin Operations

**Veto Amendment:**
```json
{
  "command": "feature",
  "feature": "AmendmentHash",
  "vetoed": true
}
```

**Remove Veto:**
```json
{
  "command": "feature",
  "feature": "AmendmentHash",
  "vetoed": false
}
```

**Admin Only:**
- Requires admin credentials
- Modifies node's voting preferences
- Persisted to database
- Takes effect immediately

***

## Operational Consequences

### Unsupported Enabled Amendments

**Detection:**
- Node checks each enabled amendment
- Compares against supported list
- Identifies unsupported active amendments

**Response:**

**setAmendmentBlocked():**
- Server logs critical error
- Sets amendment blocked flag
- Stops processing ledgers
- Prevents incorrect behavior

**Error Message:**
```
Unsupported amendment [AmendmentHash] activated.
This server is amendment blocked.
```

**Recovery Options:**

1. **Upgrade Software:**
   - Install rippled version supporting amendment
   - Restart node
   - Resume normal operation

2. **Wait for Code:**
   - If amendment very new, wait for release
   - Monitor rippled repository
   - Plan upgrade window

3. **Network Fallback:**
   - Node excluded from consensus
   - Can still query data
   - Cannot validate or propose

### Veto Power and Minority Protection

**How Veto Works:**

**Withholding Support:**
- Simply not voting "yes" is a veto
- No explicit "no" vote needed
- Absence of support blocks activation

**Minority Protection:**
- Just 21% opposition blocks amendments
- Significant minority has power
- Prevents controversial changes

**Continuous Power:**
- Can veto at any time during voting
- Even after gaining majority
- Until 2-week period completes

**Strategic Implications:**

**Conservative Bias:**
- System favors stability over change
- Higher bar for activation
- Protects against hasty decisions

**Coalition Building:**
- Proponents must build broad support
- Need 80%+ validator backing
- Encourages compromise

**Compromise Incentive:**
- Amendments designed inclusively
- Address concerns proactively
- Build consensus before proposing

***

## Edge Cases and Special Scenarios

### Low Validator Count

**Challenge:**
- Few validators make 80% easier to achieve
- Less geographic/organizational diversity
- Higher risk of coordination

**Threshold Minimum:**
- Always at least 1 vote required
- Even with 1 validator, needs support
- Prevents accidental activation

**Network Growth:**
- As validators increase, threshold rises
- More voices required for consensus
- Increases legitimacy

### Network Partitioning

**Scenario:**
- Network splits into separate groups
- Each group may vote differently
- Could activate different amendments

**Resolution:**
- When partition heals, use longest chain
- Network converges on majority view
- Minority partition abandoned

**Prevention:**
- Well-connected overlay network
- Geographic diversity
- Multiple communication paths

### Critical Bug Fixes

**Emergency Amendments:**

**Accelerated Process:**
- Validators may vote quickly
- Community coordination on urgency
- Faster than normal 2-week period

**Same Mechanism:**
- Uses normal amendment process
- No special bypass
- Just expedited timeline

**Risk Assessment:**
- Balance speed vs. safety
- Critical security issues justify haste
- Community consensus on urgency

**Process Adaptation:**
- Informal agreement to expedite
- Higher risk tolerance
- Accept some risk for urgent fixes

### Amendment Voting in Standalone Mode

**Behavior:**
- Standalone nodes don't participate in consensus
- No validator voting
- Can enable amendments manually

**Configuration:**
```
[amendments]
# Enable specific amendment in standalone
B2A4DB846F0891BF2C76AB2F2ACC8F5B4EC64437135C6E56F3F859DE5FFD5856
```

**Use Cases:**
- Testing amendment behavior
- Development and debugging
- Isolated testing environments

***

## Interactions with Other Systems

### Negative UNL Integration

**Parallel Voting:**
- Amendment voting runs alongside Negative UNL voting
- Both occur on flag ledgers
- Fee voting also concurrent

**Shared Infrastructure:**
- Same voting ledger mechanism
- Same trusted validator tracking
- Coordinated pseudo-transactions

**Independence:**
- Each votes independently
- Different thresholds and timings
- No interference between systems

### Fee Voting

**Different from Amendments:**

**Continuous Adjustment:**
- Fees can change regularly
- Every 256 ledgers opportunity

**Quantitative Decision:**
- Not just yes/no
- Specific fee values
- Multiple options

**Operational Parameter:**
- Affects network economics
- Not protocol rules
- Faster process

**Consensus Challenge:**
- Must agree on specific values
- Median or majority vote
- Economic impact considerations

***

## Summary

### Key Takeaways

- **Democratic governance** enables protocol evolution without central authority
- **80% supermajority** requirement balances progress with stability
- **Two-week sustained support** prevents hasty or temporary decisions
- **Transparent voting** recorded permanently on-ledger
- **Veto power** protects significant minorities from unwanted changes
- **Amendment blocking** protects network from unsupported activations
- **Coordinated activation** ensures network-wide consensus

### The Big Picture

The amendment system demonstrates how distributed networks can evolve and improve while maintaining consensus and avoiding the pitfalls that have affected other blockchain projects. By combining democratic voting with strong consensus requirements, sustained support periods, and transparent on-ledger tracking, XRPL achieves:

- **Continuous innovation** without compromising decentralization
- **Network stability** through careful change management
- **Minority protection** via veto power
- **Transparency** with public voting records
- **Coordination** ensuring synchronized network upgrades
- **Flexibility** allowing protocol to adapt to new requirements

This sophisticated governance mechanism is fundamental to XRPL's ability to remain competitive and relevant while maintaining its core principles of decentralization, reliability, and security.

***

## References to Source Code

- `rippled/src/xrpld/app/misc/AmendmentTable.h` - Amendment table interface
- `rippled/src/xrpld/app/misc/detail/AmendmentTable.cpp` - Amendment table implementation
- `rippled/src/xrpld/app/tx/detail/Change.cpp` - Amendment transaction application
- `rippled/src/xrpld/app/rdb/detail/Wallet.cpp` - Database persistence
- `rippled/src/xrpld/rpc/handlers/Feature1.cpp` - RPC interface
- `rippled/include/xrpl/protocol/Feature.h` - Feature definitions
- `rippled/include/xrpl/protocol/detail/features.macro` - Amendment registration
- `rippled/src/libxrpl/protocol/Feature.cpp` - Feature implementation
- `rippled/src/xrpld/consensus/ConsensusParms.h` - Consensus parameters
- `rippled/src/xrpld/app/misc/README.md` - Amendment system documentation
