# Homework 1: Validator Keys Setup and Management

[← Back to Consensus II](./)

***

### Objective

This homework provides hands-on experience with **XRPL validator key management**. You will build rippled with validator keys support, generate and manage validator keys, understand the token architecture, and practice key rotation and revocation procedures.

**Format**: Written report (PDF or Markdown) with command outputs, screenshots, and analysis.

***

### Background

XRPL validators use a **two-key system** for security:

1. **Master Key Pair**: Long-term identity keys stored offline
   - Signs tokens that authorize servers
   - Signs revocations if compromised
   - Should be kept highly secure

2. **Ephemeral Keys**: Short-term operational keys
   - Used in validator tokens
   - Can be rotated regularly
   - Automatically distributed via tokens

**Token System**:
- Each new token invalidates all previous tokens
- Tokens contain manifest and validation secret key
- Hard limit: 4,294,967,293 tokens per master key pair
- Enables key rotation without changing validator identity

***

### Task

Set up a complete validator key management system, perform key operations, and analyze the security model.

#### Requirements

1. **Build Configuration**
   * Build rippled with `validator_keys=ON` flag
   * Build the `validator-keys` tool
   * Verify successful compilation
   * Document build commands and output

2. **Initial Key Generation**
   * Generate master key pair using `validator-keys`
   * Locate and examine the `validator-keys.json` file
   * Generate first validator token
   * Document the generated public key
   * Extract and analyze token contents

3. **Token Analysis**
   * Use [XRPL Binary Visualizer](https://richardah.github.io/xrpl-binary-visualizer/)
   * Decode your validator token
   * Identify the manifest structure
   * Identify the validation secret key encoding
   * Document what information is contained in the token

4. **Configuration Integration**
   * Add `[validator_token]` to `rippled.cfg`
   * Start rippled in standalone mode
   * Verify validator is properly configured
   * Check log files for validator initialization
   * Document verification steps

5. **Key Rotation Practice**
   * Generate a second validator token
   * Update rippled configuration
   * Restart rippled
   * Verify old token is invalidated
   * Verify new token is accepted
   * Document the rotation process

6. **Security Analysis**
   * Analyze the token revocation mechanism
   * Document the `[validator_key_revocation]` format
   * Explain when revocation should be used
   * Describe post-revocation recovery steps

***

### Deliverable

A written report containing:

* **Build Documentation**:
  * CMake configuration command used
  * Build output showing successful compilation
  * validator-keys binary location

* **Key Generation Evidence**:
  * Output from `create_keys` command
  * Output from `create_token` command
  * Validator public key (format: `nH...`)
  * Token string (first 50 characters)

* **Token Analysis Table**:

  | Component | Description | Location in Token |
  |-----------|-------------|-------------------|
  | Manifest | ... | ... |
  | Validation Secret Key | ... | ... |
  | Sequence Number | ... | ... |
  | Signature | ... | ... |

* **Configuration File**:
  * Relevant sections of `rippled.cfg`
  * Show both `[validator_token]` sections (before and after rotation)

* **Log Analysis**:
  * Rippled startup logs showing validator initialization
  * Evidence of successful key rotation
  * Any warnings or errors encountered

* **Security Questions Answered**:
  1. Why use a two-key system instead of a single key?
  2. What information must remain secret vs. public?
  3. What are the consequences of losing the master private key?
  4. How does token-based rotation improve security?
  5. What happens if someone obtains an old token?
  6. When should you revoke keys vs. just rotate?

***

### Tips

* **Build Issues**: If validator_keys fails to build, check Boost version (requires 1.75+)
* **Key Storage**: The `validator-keys.json` file contains your master private key—back it up securely!
* **Token Inspection**: The XRPL Binary Visualizer helps understand token structure
* **Verification**: Use `rippled server_info` to check validator status
* **Log Location**: Check `~/.ripple/debug.log` for validation messages
* **Testing**: Standalone mode is sufficient for this homework (no network needed)

**Common Commands**:
```bash
# Build validator-keys
cmake --build . --target validator-keys --parallel 10

# Generate keys
cd validator-keys && ./validator-keys create_keys

# Generate token
./validator-keys create_token

# Check validator status
./rippled server_info | grep -A 10 validation
```

***

### Advanced Challenges (Optional)

1. **Multi-Server Setup**: Generate tokens for 3 different servers using the same master key
2. **Revocation Testing**: Practice the complete revocation workflow
3. **Token Expiry**: Investigate if there's a token expiration mechanism
4. **Automation**: Write a script to automate token rotation

***

### Learning Goals

By completing this homework, you should be able to:

* Build rippled with specialized configuration flags
* Generate and manage validator key pairs
* Understand the validator token security model
* Perform key rotation procedures
* Analyze cryptographic structures in tokens
* Explain the security benefits of the two-key system
* Handle key compromise scenarios

***

### Reference Materials

* **Validator Setup Guide**: [XRPL Validator Setup](https://xrpl.org/run-a-rippled-validator.html)
* **Key Management**: `rippled/src/xrpld/app/misc/ValidatorKeys.h`
* **Token Format**: `rippled/src/xrpl/protocol/STValidation.h`
