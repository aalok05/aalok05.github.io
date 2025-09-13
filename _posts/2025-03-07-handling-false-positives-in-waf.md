---

layout:  post

title:  "Handling False Positives in Azure Application Gateway WAF with OWASP Rules"

categories:  Cloud and Networking

author:  "Aalok Singh"

---

Azure Web Application Firewall (WAF) in Application Gateway helps protect web applications from threats by using pre-configured rules, including the 'Open Web Application Security Project' (OWASP) Core Rule Set (CRS). However, these rules can sometimes block legitimate traffic, causing false positives.

Recently, one of our clients faced an issue where their WAF was incorrectly flagging legitimate users as malicious, denying them access. This disrupted operations and highlighted the need for fine-tuning WAF settings.

## Why False Positives Happen

False positives occur when WAF misidentifies a legitimate request as an attack. Common reasons include:

1.  Strict OWASP Rules  -- OWASP CRS is aggressive by default, sometimes blocking normal traffic that resembles attack patterns.
2.  Special Characters in Input  -- If a request contains characters like < script > or SQL-like keywords, it may trigger SQL Injection or XSS rules.
3.  APIs and JSON Payloads  -- API requests with structured data (JSON/XML) can be misclassified as malicious.
4.  Sensitive Query Parameters  -- URLs with words resembling attack payloads might get blocked.
5.  Unique Application Behavior  -- Custom workflows may not align with predefined WAF rules.

## How Anomaly Scoring Works

WAF assigns a score to each request based on detected rule violations. If the cumulative anomaly score exceeds a threshold (default: 5), the request is blocked. Lowering the score threshold increases sensitivity, while raising it reduces false positives.

## Identifying False Positives

1.  Check WAF Logs  -- Logs in Azure Monitor show details of blocked requests and rule IDs.
2.  Use Detection Mode  -- Running WAF in "Detection" mode helps identify issues without blocking traffic.
3.  Analyze Blocked Requests  -- Reviewing request payloads helps pinpoint the cause of false positives.

## Fixing False Positives

### 1\. Adjust OWASP Rule Set Sensitivity

Azure WAF allows you to choose different OWASP CRS versions (e.g., 3.1, 3.2), each with varying levels of strictness. Some newer versions reduce false positives by improving attack pattern detection. If legitimate requests are frequently being blocked, consider testing a different OWASP rule set version to see if it results in fewer disruptions.

Additionally, adjusting the  anomaly scoring threshold  can help. If the WAF is blocking too many valid requests, increasing the anomaly score threshold can allow more traffic through while still preventing actual threats. Conversely, lowering the threshold can make WAF more aggressive in blocking suspicious activity.

### 2\. Create Custom Rules

In some cases, predefined WAF rules may not align with specific application behavior. Creating  custom rules  allows for more flexible traffic control. Here's how:

-   Allow trusted IP ranges for internal applications or API requests.
-   Bypass specific headers, query parameters, or request bodies that are known to trigger false positives.
-   Define rules to exclude requests from evaluation based on user roles or authentication status.

Custom rules should be tested in  Detection mode  before enforcing them to ensure they don't introduce vulnerabilities.

### 3\. Tune Request Body Inspection

By default, WAF inspects request bodies to detect malicious payloads, but this can sometimes lead to false positives when processing large JSON or XML payloads. To fine-tune this:

-   Increase the  request body size limit  if valid requests are getting blocked due to payload size.
-   Configure  content type exclusions  to prevent unnecessary inspections for non-critical endpoints.
-   Modify  request body inspection thresholds  to ensure only high-risk content gets flagged.

### 4\. Test with Detection Mode

Run WAF in Detection mode before enforcing strict blocking to observe behavior. Detection mode helps analyze the impact of WAF rules without actually blocking traffic

## Conclusion

A key takeaway for me in the recent events at one of our clients was - there cannot be a 'checkbox' approach to security, where we enable OWASP rules and forget, assuming everything is now secure.

WAF rule sets are a tool which we have to fine tune such that it strikes the right balance between protection and usability, as per your unique needs.