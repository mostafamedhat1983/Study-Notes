
## Overview

  

This guide explains how to request an SSL/TLS certificate from AWS Certificate Manager (ACM) and validate it using DNS records in Namecheap. This certificate enables HTTPS for your application exposed via AWS Application Load Balancer (ALB).

  

---

  

## Prerequisites

  

- AWS Account with access to Certificate Manager

- Domain registered (e.g., `mostafa-medhat.online` on Namecheap)

- Access to domain DNS settings

  

---

  

## Part 1: Request ACM Certificate

  

### Step 1: Navigate to AWS Certificate Manager

  

1. Log in to **AWS Console**

2. **IMPORTANT:** Select the **correct region** (must match your EKS cluster region)

   - For this project: **us-east-2 (Ohio)**

   - **Why it matters:** ALB can only use certificates from the same region

3. Search for **"Certificate Manager"** or **"ACM"**

4. Click **AWS Certificate Manager**

  

---

  

### Step 2: Request a Public Certificate

  

1. Click **"Request a certificate"** (orange button)

2. Select **"Request a public certificate"** (default)

3. Click **"Next"**

  

---

  

### Step 3: Enter Domain Names

  

#### Single Subdomain (Basic):

```

chatbot.mostafa-medhat.online

```

  

#### Wildcard Certificate (Recommended):

```

chatbot.mostafa-medhat.online

*.mostafa-medhat.online

```

  

**Why add wildcard?**

- Same certificate works for any subdomain (jenkins, api, admin, etc.)

- Free with ACM (no additional cost)

- Simplifies future deployments

  

**Example use cases:**

- `chatbot.mostafa-medhat.online` → Chatbot frontend

- `jenkins.mostafa-medhat.online` → Jenkins UI

- `api.mostafa-medhat.online` → API endpoints

  

Click **"Add another name to this certificate"** to add the wildcard.

  

---

  

### Step 4: Select DNS Validation Method

  

1. Choose **"DNS validation - recommended"**

  

**DNS Validation vs Email Validation:**

  

| Feature | DNS Validation ✅ | Email Validation |

|---------|------------------|------------------|

| Automatic renewal | Yes | No |

| Domain email required | No | Yes |

| Works with any registrar | Yes | Yes |

| Setup complexity | One-time DNS records | Email verification each time |

| Best for | Production, automation | Quick testing only |

  

2. Click **"Next"**

  

---

  

### Step 5: Add Tags (Optional but Recommended)

  

Tags help organize resources in AWS:

  

```

Key: Project       Value: chatbot

Key: Environment   Value: dev

Key: ManagedBy     Value: terraform

```

  

Click **"Next"**

  

---

  

### Step 6: Review and Request

  

**Review your settings:**

- Domain names: `chatbot.mostafa-medhat.online` and `*.mostafa-medhat.online`

- Validation method: DNS validation

- Region: us-east-2

  

Click **"Request"**

  

**Success message:** "Certificate request submitted"

  

**Status:** "Pending validation" (yellow banner)

  

---

  

## Part 2: DNS Validation with Namecheap

  

### Step 1: Get DNS Validation Records from ACM

  

1. Click **"View certificate"** or select your certificate from the list

2. Certificate status shows: **"Pending validation"**

3. Scroll to **"Domains"** section

  

**You'll see validation records for EACH domain:**

  

#### Example for `chatbot.mostafa-medhat.online`:

```

CNAME Name:  _a1b2c3d4e5f6.chatbot.mostafa-medhat.online

CNAME Value: _x1y2z3a4b5c6.acm-validations.aws.

```

  

#### Example for `*.mostafa-medhat.online`:

```

CNAME Name:  _g7h8i9j0k1l2.mostafa-medhat.online

CNAME Value: _m3n4o5p6q7r8.acm-validations.aws.

```

  

**IMPORTANT:**

- Each domain has **different** validation records

- Keep this page open - you'll need these values

- The trailing dot (`.`) may or may not be needed (Namecheap doesn't need it)

  

---

  

### Step 2: Log in to Namecheap

  

1. Go to **namecheap.com**

2. Log in to your account

3. **Dashboard** → **Domain List**

4. Find **mostafa-medhat.online**

5. Click **"Manage"**

  

---

  

### Step 3: Access DNS Settings

  

1. Click **"Advanced DNS"** tab

2. You'll see existing DNS records (likely Vercel records for your CV)

  

**DO NOT DELETE EXISTING RECORDS** - We're adding new ones, not replacing.

  

---

  

### Step 4: Add CNAME Records for Validation

  

#### For chatbot.mostafa-medhat.online:

  

1. Click **"Add New Record"**

2. **Type:** Select **"CNAME Record"**

3. **Host:**

   - ACM shows: `_a1b2c3d4e5f6.chatbot.mostafa-medhat.online`

   - **Enter in Namecheap:** `_a1b2c3d4e5f6.chatbot`

   - **Why?** Namecheap auto-appends `.mostafa-medhat.online`

4. **Value:**

   - Copy from ACM: `_x1y2z3a4b5c6.acm-validations.aws.`

   - **Remove trailing dot:** `_x1y2z3a4b5c6.acm-validations.aws`

5. **TTL:** Select **"Automatic"** or **"1 min"** (faster validation)

6. Click **green checkmark** to save

  

#### For *.mostafa-medhat.online (if added):

  

1. Click **"Add New Record"**

2. **Type:** Select **"CNAME Record"**

3. **Host:**

   - ACM shows: `_g7h8i9j0k1l2.mostafa-medhat.online`

   - **Enter in Namecheap:** `_g7h8i9j0k1l2`

4. **Value:** `_m3n4o5p6q7r8.acm-validations.aws`

5. **TTL:** Automatic

6. Click **green checkmark**

  

---

  

### Step 5: Verify DNS Records Added

  

**Your Namecheap DNS should now show:**

  

| Type | Host | Value | TTL |

|------|------|-------|-----|

| CNAME | _a1b2c3d4e5f6.chatbot | _x1y2z3a4b5c6.acm-validations.aws | Automatic |

| CNAME | _g7h8i9j0k1l2 | _m3n4o5p6q7r8.acm-validations.aws | Automatic |

| ... | (your existing Vercel records) | ... | ... |

  

**CRITICAL: Common Mistakes to Avoid:**

  

❌ **DON'T** enter full domain in Host field:

```

Wrong: _abc123.chatbot.mostafa-medhat.online

Right: _abc123.chatbot

```

  

❌ **DON'T** include trailing dot in Value:

```

Wrong: _xyz789.acm-validations.aws.

Right: _xyz789.acm-validations.aws

```

  

❌ **DON'T** delete existing DNS records (Vercel, etc.)

  

---

  

## Part 3: Wait for Certificate Validation

  

### Expected Timeline:

  

- **0-5 minutes:** DNS records propagate from Namecheap

- **5-15 minutes:** AWS detects DNS records

- **15-30 minutes:** Certificate validated and issued

- **Rare cases:** Up to 72 hours if DNS propagation is slow

  

---

  

### Monitor Validation Status:

  

1. Return to **AWS Certificate Manager**

2. Refresh the certificate page (F5)

3. Check **"Status"** column

  

**Status progression:**

- `Pending validation` (yellow) → Still waiting

- `Issued` (green) → ✅ **Success!**

  

---

  

### Troubleshooting Validation Delays

  

#### If status is still "Pending validation" after 1 hour:

  

**1. Verify DNS Records in Namecheap:**

- Check Host field format: Should be `_abc123.chatbot`, NOT full domain

- Check Value field: No trailing dot

- Ensure record is saved (green checkmark visible)

  

**2. Test DNS Propagation:**

  

Use online tool: https://dnschecker.org

- Enter: `_a1b2c3d4e5f6.chatbot.mostafa-medhat.online`

- Type: CNAME

- Click "Search"

- Should show validation value globally

  

**3. Check AWS Console for Errors:**

- Certificate details page may show specific error messages

  

**4. Common Issues:**

| Issue | Solution |

|-------|----------|

| DNS not propagating | Wait 30 more minutes, then check dnschecker.org |

| Wrong Host format | Re-enter as `_abc123.chatbot` (not full domain) |

| Wrong Value | Copy exactly from ACM, remove trailing dot |

| TTL too high | Change to "Automatic" or "1 min" |

  

---

  

## Part 4: Get Certificate ARN

  

### Once Certificate Status Shows "Issued":

  

1. In **AWS Certificate Manager**, click on your certificate

2. At the top, find **"ARN"** (Amazon Resource Name)

  

**Example ARN:**

```

arn:aws:acm:us-east-2:123456789012:certificate/a1b2c3d4-5678-90ab-cdef-1234567890ab

```

  

3. Click the **copy icon** next to ARN

4. **Save this ARN** - You'll need it for:

   - Terraform Ingress annotation

   - Helm values configuration

  

---

  

## Key Concepts Explained

  

### What is DNS Validation?

  

DNS validation proves you own the domain by:

1. AWS gives you unique CNAME records

2. You add those records to your DNS provider

3. AWS checks if records exist

4. If found, AWS knows you control the domain

  

**Analogy:** AWS asks you to put a specific sign in your yard. If they see it, they know it's your house.

  

---

  

### Why Use Subdomain (chatbot.mostafa-medhat.online)?

  

**Benefits:**

- ✅ Keep existing setup (CV on Vercel) unchanged

- ✅ Clean separation of projects

- ✅ Professional structure

- ✅ Easy to add more projects later (api.*, jenkins.*)

  

**DNS Structure:**

```

mostafa-medhat.online           → Vercel (CV)

chatbot.mostafa-medhat.online   → AWS ALB (Chatbot)

jenkins.mostafa-medhat.online   → Future: Jenkins

api.mostafa-medhat.online       → Future: API

```

  

---

  

### Certificate Renewal

  

**ACM automatically renews certificates if DNS validation records remain:**

- ✅ **DO NOT DELETE** the CNAME validation records from Namecheap

- ✅ AWS checks records periodically before expiration

- ✅ Renewal happens automatically (60 days before expiry)

- ✅ No action needed from you

  

---

  

## Security Best Practices

  

1. ✅ **Always use DNS validation** (not email)

2. ✅ **Keep validation records in DNS** (enables auto-renewal)

3. ✅ **Use wildcard certificates** for multiple subdomains

4. ✅ **Certificate in same region as ALB** (us-east-2)

5. ✅ **Tag certificates** for organization

  

---

  

## Cost

  

**AWS Certificate Manager (ACM):**

- ✅ **FREE** for public certificates

- ✅ **FREE** auto-renewal

- ✅ **FREE** wildcard certificates

- ✅ **FREE** unlimited subdomains

  

**You only pay for:**

- ALB usage (~$16/month)

- Data transfer through ALB

  

---

  

## Next Steps

  

After getting certificate ARN:

1. ✅ Proceed to deployment guide (`02-alb-ingress-deployment.md`)

2. Add AWS Load Balancer Controller to Terraform

3. Create Ingress resource with certificate ARN

4. Point domain CNAME to ALB DNS name

  

---

  

## Reference Commands

  

### Check DNS Propagation (Linux/Mac/WSL):

```bash

# Check CNAME record

dig _a1b2c3d4e5f6.chatbot.mostafa-medhat.online CNAME

  

# Check with specific DNS server

nslookup -type=CNAME _a1b2c3d4e5f6.chatbot.mostafa-medhat.online 8.8.8.8

```

  

### Windows PowerShell:

```powershell

Resolve-DnsName -Name _a1b2c3d4e5f6.chatbot.mostafa-medhat.online -Type CNAME

```

  

---

  

## Summary Checklist

  

- [ ] ACM certificate requested in correct region (us-east-2)

- [ ] DNS validation method selected

- [ ] CNAME records added to Namecheap (correct Host format)

- [ ] DNS propagation verified (dnschecker.org)

- [ ] Certificate status changed to "Issued"

- [ ] Certificate ARN copied and saved

- [ ] Validation records kept in DNS (for auto-renewal)

  

---

  

**Document Version:** 1.0  

**Last Updated:** 2025-12-02  

**Project:** AI Chatbot Platform  

**Domain:** mostafa-medhat.online