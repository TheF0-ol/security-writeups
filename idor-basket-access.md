---
title: "IDOR in cart basket access endpoints"
slug: "idor-basket-access"
authors: ["Shravan Kakkar (TheF0-ol)"]
date: "2026-10-15"
tags: ["Web AppSec", "IDOR", "Broken Access Control", "API Testing"]
summary: "Real-world IDOR in cart access: basket ID in request body not validated against authenticated identity. Full PoC, impact, root cause and fix."
---

## 1. Overview

- Objective: Document the IDOR finding submitted to the MyProtein Bugcrowd bounty program and convert the submission into a portfolio writeup.
- What the app did: The AddToBasket and UpdateBasketQuantity API endpoints identified the user via X-Chumewe-User / X-Chumewe-Session headers, but did not verify that the basket ID supplied in the request body actually belonged to that identity. An authenticated attacker could manipulate another shopper's basket by supplying the victim's basket ID.
- What I did: Created two separate accounts, added an item to Account A's basket, noted the basket ID from the AddToBasket response, then as Account B sent AddToBasket and UpdateBasketQuantity requests with Account A's basket ID. Observed Account A's basket ID returned with "merged": true and confirmed Account A's basket reflected the unauthorized changes.

## 2. Target details

- Target: (Bugcrowd program, Web App, VRT)
- Vulnerability class: Broken Access Control > IDOR > Modify Sensitive Information (Iterable Object Identifiers)
- Priority: N/A
- My setup: Two accounts created for this purpose (Account A and Account B). No real customer data was accessed or modified.

## 3. My findings

- OWASP Top 10 2021: A01:2021 - Broken Access Control
- CWE: CWE-639 (Authorization Bypass Through User-Controlled Key)
- Risk rating: Medium / High (P2)
- Impact: Any authenticated user could add items, change quantities and change prices in another shopper's basket. This undermines trust in the checkout process and affects purchase decisions.

## 4. Tools I used

- Browser / Bugcrowd interface - to create test accounts and submit the finding
- Burp Suite Community (Proxy/Intercept) - to capture and modify the AddToBasket and UpdateBasketQuantity requests
- Burp Repeater - to send requests with modified basket IDs
- Two browser sessions - Account A and Account B

## 5. How I found it

### 5.1 Discovery

1. Scoped the Web App VRT for Broken Access Control / IDOR / Insecure Direct Object References.
2. Noted the basket ID format from Account A's AddToBasket response.
3. Tested the Cookie header for authentication on the basket endpoints — confirmed it had no bearing.
4. Tested the X-Chumewe-User / X-Chumewe-Session headers as the identified identity, but found neither endpoint verified that the basket ID in the request body matched that identity.

### 5.2 Exploitation

5. Sent POST /api/operation/AddToBasket/ as Account B with Account A's basket ID in the variables.id field.
6. Response returned Account A's basket ID with "merged": true, confirming the item was added to Account A's basket.
7. Sent POST /api/operation/UpdateBasketQuantity/ as Account B with Account A's basket ID, changing the quantity and price of an item in Account A's basket.
8. Confirmed the changes directly by logging into Account A's account.

### 5.3 Confirming the chain

9. Account B's session was used for the request, but the basket ID belonged to Account A. The endpoints did not validate this mismatch.

## 6. Proof of concept

### 6.1 Request

```
POST /api/operation/AddToBasket/ HTTP/2
Host: www.myprotein.com
X-Chumewe-User: <Account B's valid X-Chumewe-User>
X-Chumewe-Session: <Account B's valid X-Chumewe-Session>
Content-Type: text/plain;charset=UTF-8

{"operationName":"AddToBasket","variables":{"id":"<victim's basket id>","items":[{"sku":"<item SKU>","quantity":1}],"currency":"GBP","shippingDestination":"GB","loyaltyEnabled":false}}
```

### 6.2 Response (redacted)

```
HTTP/1.1 200 OK
Content-Type: application/json
...

{"basketId":"<victim's basket id>","merged":true,...}
```

What I noticed:
- No authorization check on the basket ID in the request body. The authenticated identity and the supplied basket ID did not need to match.
- The Cookie header alone did not drive authorization; the X-Chumewe-User / X-Chumewe-Session headers did, but the server did not validate the basket ownership.

## 7. Impact

- Authentication: Any authenticated user could modify another user's basket.
- Data exposure: Another shopper's basket contents were accessible and modifiable.
- Purchase integrity: Account B could add items, change quantities, and change prices in Account A's basket, directly ahead of a purchase decision.
- Customer trust: Unauthorized basket modification undermines trust in the integrity of the checkout process.

## 8. Fix

1. Derive the basket server-side from the authenticated session/identity, not from client-supplied input.
2. If a client-supplied basket ID must be accepted, validate it against the requester's identity before applying the mutation. Reject mismatches.
3. Apply the same ownership check to both AddToBasket and UpdateBasketQuantity.

## 9. What I learned

- IDOR is not about guessable IDs. A UUID-supplied object reference is still an object reference. The vulnerability is when the server trusts the client-supplied identifier instead of deriving it from the authenticated session.
- Always test the authorization boundary, not just the object ID format.
- The headers/cookies that authenticate you are not necessarily the ones that authorize you. The authorization check is a separate step.
- Implement server-side ownership validation for all object references, even if they look opaque.
- Root cause and fix are the most important sections for a real-world writeup. They show you understand the code-level reason and the correct remediation.

## 10. References

- Bugcrowd: MyProtein program - Web App VRT, Broken Access Control > IDOR > Modify Sensitive Information
- OWASP Top 10:2021 - A01:2021 Broken Access Control
- MITRE CWE-639 - Authorization Bypass Through User-Controlled Key

## 11. Notes

- Testing was performed only between two accounts created by the researcher for this purpose. No real customer data was accessed or modified at any point.
- This writeup documents my finding and methodology. It is intended for portfolio and interview prep purposes.
