# 🛠️ ZAP Scanner Overview
ZAP (Zed Attack Proxy) includes a web scanner like Burp. It can:

- Build a site map (with Spider)

- Do Passive Scans (no attacks)

- Do Active Scans (send attack payloads)

# 🕷️ Spider (Site Crawler)
Finds pages by following links.
Start it by:

- Right-clicking a request → Attack > Spider

- Or clicking the Spider button in HUD (browser view).

- If prompted, allow ZAP to add the site to scope.

- Results appear in Sites Tree.

- 👉 Ajax Spider: Also finds content loaded dynamically via JavaScript (AJAX).

# 🛡️ Passive Scanner
Runs automatically during Spider.
Looks for:

- Missing security headers

- Potential XSS or DOM-based issues

- Alerts appear:

- Left: Page-specific

- Right: Whole site

# 🔍 Active Scanner
Start it after Spider completes (or ZAP will spider first).
Sends attacks to all pages/inputs to find real vulnerabilities.

- Takes more time, shows progress and alerts.

Can detect:

- XSS

- SQLi

- Command injection

# 👀 Example Alert:

- High Risk: Remote OS Command Injection

- Evidence: /etc/passwd output found

You can:

- Click alerts for details

- Replay requests

- Use ZAP HUD or request editor

# 📝 Reporting
Generate reports: Reports > Generate HTML Report

Save as HTML, XML, or Markdown

Shows all alerts: High, Medium, Low, Informational

Useful for documentation or bug reports
