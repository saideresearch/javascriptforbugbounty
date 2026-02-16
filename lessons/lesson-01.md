
# 1. Why Variables Matter in Bug Bounty

In real web applications, JavaScript variables often store:

* Authentication tokens
* User roles
* Feature flags
* Pricing values
* API endpoints
* Permission flags
* Sensitive configuration data

If a developer relies on these variables for security decisions on the client side, that can lead to vulnerabilities.

Important principle:

Client-side JavaScript is fully controlled by the attacker.

---

# 2. `var` — Function Scoped and Globally Attached

## Behavior

* Function-scoped
* Hoisted
* Can be redeclared
* If declared globally, becomes a property of `window`

Example:

```javascript
var isAdmin = false;
```

If this is in global scope, it becomes:

```javascript
window.isAdmin
```

---

## Meaningful Bug Bounty Example: UI-Based Authorization Bypass

Imagine the application code:

```javascript
var isAdmin = false;

if (isAdmin) {
    document.getElementById("adminPanel").style.display = "block";
}
```

The developer assumes that only admins will see the admin panel.

As an attacker, open DevTools and run:

```javascript
isAdmin = true;
document.getElementById("adminPanel").style.display = "block";
```

Now the admin panel appears.

If the backend also fails to verify authorization properly, you may:

* Access admin APIs
* Trigger privileged actions
* Escalate privileges

Even if the backend is secure, this may expose hidden endpoints, internal functionality, or sensitive data.

---

## Risk 1: Global Scope Pollution

Because `var` attaches to `window`:

```javascript
var apiBase = "https://api.example.com";
```

You can override it:

```javascript
apiBase = "https://attacker.com";
```

If the application dynamically builds requests using this variable, it may:

* Send data to an attacker-controlled endpoint
* Leak tokens
* Enable data exfiltration

---

## Risk 2: Hoisting Causing Logic Errors

Example:

```javascript
if (isPremiumUser) {
    enablePremiumFeatures();
}

var isPremiumUser = false;
```

Because `var` is hoisted, this becomes internally:

```javascript
var isPremiumUser;

if (isPremiumUser) {
    enablePremiumFeatures();
}

isPremiumUser = false;
```

If some earlier script accidentally assigns a value before this block runs, the logic may behave unexpectedly.

Hoisting-related bugs sometimes cause authorization logic to fail in edge cases.

---

# 3. `let` — Block Scoped but Still Mutable

## Behavior

* Block-scoped
* Not attached to `window`
* Cannot be redeclared in same scope
* Can be reassigned

Example:

```javascript
let price = 100;
```

---

## Meaningful Bug Bounty Example: Client-Side Price Manipulation

Suppose a checkout page contains:

```javascript
let price = 100;
let discount = 10;

let finalPrice = price - discount;

document.getElementById("total").innerText = finalPrice;
```

When user clicks "Pay":

```javascript
fetch("/api/pay", {
    method: "POST",
    body: JSON.stringify({
        amount: finalPrice
    })
});
```

If the backend trusts the client-provided amount, you can:

Open console:

```javascript
discount = 100;
finalPrice = 0;
```

Then click pay.

If the backend does not validate the price independently, this is a critical vulnerability (business logic flaw).

---

## Important Insight

Even though `let` is not attached to `window`, you can still modify it if it is accessible in the current execution scope.

Never assume `let` provides security.

---

# 4. `const` — Constant Reference, Not Constant Data

## Behavior

* Block-scoped
* Cannot be reassigned
* Objects and arrays inside are still mutable

Example:

```javascript
const user = {
    role: "user"
};
```

You cannot do:

```javascript
user = {};
```

But you can do:

```javascript
user.role = "admin";
```

---

## Meaningful Bug Bounty Example: Role Escalation via Object Mutation

Consider this code:

```javascript
const currentUser = {
    id: 123,
    role: "user"
};

function canDeleteAccount() {
    return currentUser.role === "admin";
}
```

Before triggering a sensitive action:

```javascript
if (canDeleteAccount()) {
    deleteAccount();
}
```

As attacker:

```javascript
currentUser.role = "admin";
```

Now:

```javascript
canDeleteAccount();
```

Returns true.

If the frontend now allows you to call privileged API endpoints without backend validation, you have privilege escalation.

---

# 5. Real-World Pattern You Will See

Very common in production apps:

```javascript
const config = {
    apiUrl: "https://api.example.com",
    enableDebug: false,
    isInternalUser: false
};
```

As attacker:

```javascript
config.enableDebug = true;
config.isInternalUser = true;
```

You may unlock:

* Debug endpoints
* Internal features
* Additional API parameters
* Verbose error messages

Debug mode sometimes exposes sensitive information.

---

# 6. Variables and Token Exposure

Example:

```javascript
var authToken = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...";
```

If token is stored like this:

* It is accessible via console
* It is readable by XSS
* It can be stolen

Check:

```javascript
window.authToken
```

If accessible, that is a security weakness.

Secure applications store tokens in HttpOnly cookies, not in JavaScript variables.

---

# 7. Key Bug Bounty Mindset

When reviewing JavaScript files:

Search for:

* `var`
* `let`
* `const`

Identify:

* Role checks
* Feature flags
* Boolean access controls
* API endpoints
* Pricing logic
* Token storage
* Debug flags

Then test:

* Can I modify this variable?
* Does the UI change?
* Does the request payload change?
* Does the backend validate independently?

---

# 8. Critical Rule

Changing a JavaScript variable does not break real security — unless the backend trusts the client.

Most high-impact bugs happen when:

Frontend logic is treated as a security boundary.

Your job is to find where developers made that mistake.
