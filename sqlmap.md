# Basic usage of SQLMap:


`sqlmap -u "http://www.example.com/vuln.php?id=1" --batch`

Tests if the parameter id is dynamic.


# 🔍 Common Log Messages & Their Meaning:

**"Target URL content is stable"**
Page output doesn’t change between requests — useful for detecting differences caused by SQLi payloads.

**"GET parameter 'id' appears to be dynamic"**
The id parameter changes the page response → good sign it’s linked to a database.

**"Parameter might be injectable"**
Heuristic tests suggest SQLi is possible (e.g., error message from MySQL).

**"Might be vulnerable to XSS"**
Quick check hints the parameter could be vulnerable to XSS (not SQLi-related but helpful info).

**"Back-end DBMS is 'MySQL'"**
SQLMap thinks the database is MySQL. You can skip testing other DBMS types.

**"Include all tests for 'MySQL' extending level/risk values"**
Option to run deeper tests specific to MySQL.

**"Reflective value(s) found and filtering out"**
SQLMap found the payload echoed back in the response — it filters it to avoid false positives.

**"Parameter appears to be injectable"**
SQLMap found a likely injection point using a specific technique (e.g., boolean-based).

**"--string='luther'"**
SQLMap used a known string in the response to detect true vs false conditions.

**"Time-based comparison requires a larger statistical model"**
SQLMap is measuring delays to detect time-based SQLi — needs more samples.

**"Extending UNION query tests"**
Since one technique worked, SQLMap increases test range for UNION injection.

**"'ORDER BY' technique appears to be usable"**
SQLMap uses this to quickly find the correct number of columns.

**"GET parameter 'id' is vulnerable"**
Confirmed SQL injection vulnerability. SQLMap asks if you want to keep testing other parameters.

**"Identified the following injection point(s)"**
Final list of all injection types and payloads that worked — proof of SQLi.

**"Fetched data logged to text files..."**
Output is saved locally in a folder for the target site. You can reuse this for future scans."GET parameter 'id' is vulnerable"

