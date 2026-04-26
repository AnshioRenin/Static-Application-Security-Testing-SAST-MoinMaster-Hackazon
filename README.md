# 🔐 Static Application Security Testing (SAST) — MoinMaster & Hackazon

![SonarQube](https://img.shields.io/badge/SonarQube-Community-4E9BCD?style=for-the-badge&logo=sonarqube&logoColor=white)
![Semgrep](https://img.shields.io/badge/Semgrep-Static%20Analysis-1B2D3C?style=for-the-badge)
![Docker](https://img.shields.io/badge/Docker-Containerised-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Python](https://img.shields.io/badge/Python-PHP-3776AB?style=for-the-badge&logo=python&logoColor=white)
![OWASP](https://img.shields.io/badge/OWASP-Top%2010-000000?style=for-the-badge)

> **Module:** Application Security | Dublin Business School  
> SAST assessment of two web applications — MoinMaster and Hackazon — using SonarQube and Semgrep. Both tools identified critical vulnerabilities including SQL injection, XSS, hardcoded credentials, and command injection.

---

## 📌 Overview

This project demonstrates the use of two industry-standard **Static Application Security Testing (SAST)** tools to identify security vulnerabilities in source code **before deployment** — when fixes are cheapest and most effective.

Two deliberately vulnerable web applications were scanned:

| Application | Language | Purpose |
|-------------|----------|---------|
| **MoinMaster** | Python + JavaScript | Advanced MoinMoin wiki engine implementation |
| **Hackazon** | PHP + JavaScript | Intentionally vulnerable e-commerce application for security training |

Both applications were scanned using **SonarQube** (deep code quality + security analysis) and **Semgrep** (fast, targeted vulnerability detection) — results were compared for coverage, accuracy, and false positive rates.

---

## 🎯 Objectives

- Identify security vulnerabilities before production deployment
- Compare SonarQube vs Semgrep in terms of detection coverage and false positive rates
- Produce actionable remediation guidance for each finding
- Demonstrate a dual-tool SAST methodology for CI/CD integration

---

## 🛠️ Tools & Setup

### SonarQube (Community Edition)
SonarQube is an open-source platform for continuous code quality and security scanning. It analyses 27+ programming languages and classifies issues by severity (Blocker, Critical, Major, Minor, Info) and type (Bug, Vulnerability, Code Smell).

**Setup via Docker Compose:**
```yaml
version: '3.8'
services:
  sonarqube:
    image: sonarqube:community
    container_name: sonarqube
    ports:
      - "9000:9000"
    environment:
      - SONAR_JDBC_URL=jdbc:postgresql://db:5432/sonar
      - SONAR_JDBC_USERNAME=sonar
      - SONAR_JDBC_PASSWORD=sonar
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
    depends_on:
      - db

  db:
    image: postgres:13
    container_name: sonarqube_db
    environment:
      - POSTGRES_USER=sonar
      - POSTGRES_PASSWORD=sonar
      - POSTGRES_DB=sonar
    volumes:
      - postgresql_data:/var/lib/postgresql/data
```

```bash
# Start SonarQube
docker-compose up -d

# Access dashboard
# URL: http://localhost:9000
# Default credentials: admin/admin (change on first login)
```

---

### Semgrep
Semgrep (Semantic Grep) is a lightweight, fast, open-source static analysis tool supporting 30+ languages. It uses YAML-based pattern-matching rules and the Semgrep Registry for thousands of pre-built security rules.

```bash
# Install
pip install semgrep

# Verify
semgrep --version

# Run scan with auto-config
semgrep --config=auto /path/to/application
```

---

## 📊 Results Summary

### MoinMaster — SonarQube
| Metric | Result |
|--------|--------|
| Security Rating | **E (worst possible)** |
| Bugs | 11 |
| Vulnerabilities | 13 |
| Code Smells | 148 |

### Hackazon — SonarQube
| Metric | Result |
|--------|--------|
| Security Rating | **D** |
| Reliability Rating | **D** |
| Bugs | **204** |
| Vulnerabilities | **121** |
| Code Smells | **2,062** |

---

## 🚨 Critical Findings

### 1. SQL Injection (Both Applications)

**MoinMaster (Python):**
```python
# VULNERABLE
query = "SELECT * FROM users WHERE username='" + username + "'"
cursor.execute("SELECT * FROM users WHERE name LIKE '%" + search_term + "%'")

# FIXED — use parameterized queries
query = "SELECT * FROM users WHERE username = ?"
cursor.execute(query, (username,))
```

**Hackazon (PHP):**
```php
// VULNERABLE
$query = "SELECT * FROM products WHERE category_id = " . $_GET['category'];
$sql = "SELECT * FROM users WHERE username = '{$_POST['username']}' AND password = '{$_POST['password']}'";

// FIXED
$stmt = $pdo->prepare("SELECT * FROM products WHERE category_id = ?");
$stmt->execute([$_GET['category']]);
```
**Impact:** Unauthorized data access, data manipulation, complete database compromise.

---

### 2. Cross-Site Scripting — XSS (Both Applications)

**Hackazon (PHP):**
```php
// VULNERABLE
echo "<div>" . $_POST['comment'] . "</div>";
echo "<h2>Results for: " . $_GET['search'] . "</h2>";

// FIXED
echo "<div>" . htmlspecialchars($_POST['comment'], ENT_QUOTES, 'UTF-8') . "</div>";
```
**Impact:** Session hijacking, credential theft, malicious actions performed with user privileges.

---

### 3. Weak Password Hashing (MoinMaster)

```python
# VULNERABLE — MD5 is broken
password_hash = md5(password)
hash = hashlib.md5(password.encode()).hexdigest()

# FIXED — use bcrypt or argon2
import bcrypt
password_hash = bcrypt.hashpw(password.encode(), bcrypt.gensalt())
```
**Impact:** Password hashes are trivially reversible with modern hardware — account compromise and credential stuffing attacks.

---

### 4. Hardcoded Credentials (Hackazon)

```php
// VULNERABLE — credentials exposed in source code
$db = new Database('localhost', 'root', 'password123', 'hackazon');

// FIXED — use environment variables
$db = new Database(
    getenv('DB_HOST'),
    getenv('DB_USER'),
    getenv('DB_PASS'),
    getenv('DB_NAME')
);
```
**Impact:** Full database access if source code is leaked or repository is public.

---

### 5. Command Injection (Hackazon)

```php
// VULNERABLE — user input passed directly to system()
system("ping " . $_POST['server']);

// FIXED — validate and escape input
$server = escapeshellarg($_POST['server']);
system("ping " . $server);
```
**Impact:** Arbitrary system command execution — complete server compromise.

---

### 6. Cross-Site Request Forgery — CSRF (Both Applications)

**Issue:** Form submissions that change server state were missing CSRF tokens, allowing attackers to trick authenticated users into performing unintended actions.

```php
// FIXED — generate and validate CSRF token
session_start();
$_SESSION['csrf_token'] = bin2hex(random_bytes(32));

// In form:
echo '<input type="hidden" name="csrf_token" value="' . $_SESSION['csrf_token'] . '">';

// On submission:
if (!hash_equals($_SESSION['csrf_token'], $_POST['csrf_token'])) {
    die('CSRF token mismatch');
}
```

---

### 7. Unvalidated File Inclusion (MoinMaster)

```php
// VULNERABLE — remote file inclusion via user input
include($_GET['module'] . '.php');

// FIXED — whitelist allowed modules
$allowed = ['home', 'profile', 'settings'];
if (in_array($_GET['module'], $allowed)) {
    include($_GET['module'] . '.php');
}
```
**Impact:** Remote code execution and full system compromise.

---

### 8. Insecure Direct Object Reference — IDOR (MoinMaster)

```python
# VULNERABLE — no authorization check
get_user_data(user_id)  # user_id comes directly from request

# FIXED — verify ownership before returning data
def get_user_data(user_id, requesting_user):
    if user_id != requesting_user.id and not requesting_user.is_admin:
        raise PermissionError("Unauthorized access")
    return db.query(user_id)
```

---

## 🔍 Tool Comparison

| Feature | SonarQube | Semgrep |
|---------|-----------|---------|
| False positive rate | Higher (especially code smells) | Lower — more precise |
| Setup complexity | High (requires Docker + DB) | Simple (`pip install`) |
| Speed | Slower — deep analysis | Fast — suitable for pre-commit hooks |
| Code quality metrics | ✅ Full metrics (debt, duplication, coverage) | ❌ Security-focused only |
| CI/CD integration | Best for pipeline-level gates | Best for developer workflow |
| Unique findings | Technical debt, resource leaks, architecture issues | Framework-specific issues, business logic flaws, security misconfigs |
| Best use | Long-term code quality monitoring | Fast, targeted security scanning |

### Recommendation
Use **both tools together**:
- **Semgrep** → pre-commit hooks and developer-level scanning (fast feedback)
- **SonarQube** → CI/CD pipeline gates and long-term quality monitoring (deep analysis)

---

## 📋 Vulnerability Count by Application

### MoinMaster — Full Breakdown
| Tool | Finding Type | Count |
|------|-------------|-------|
| SonarQube | Bugs | 11 |
| SonarQube | Vulnerabilities | 13 |
| SonarQube | Code Smells | 148 |
| Semgrep | File Inclusion | Multiple |
| Semgrep | SQL Injection | Multiple |
| Semgrep | Weak Hashing (MD5) | Multiple |
| Semgrep | CSRF Missing Tokens | Multiple |

### Hackazon — Full Breakdown
| Tool | Finding Type | Count |
|------|-------------|-------|
| SonarQube | Bugs | 204 |
| SonarQube | Vulnerabilities | 121 |
| SonarQube | Code Smells | 2,062 |
| Semgrep | SQL Injection | Multiple |
| Semgrep | XSS | Multiple |
| Semgrep | Hardcoded Credentials | Multiple |
| Semgrep | Command Injection | Multiple |
| Semgrep | Insecure File Upload | Multiple |
| Semgrep | Information Disclosure | Multiple |

---

## 🛡️ Remediation Priority

| Priority | Vulnerability | Applications Affected |
|----------|--------------|----------------------|
| 🔴 Immediate | SQL Injection | Both |
| 🔴 Immediate | Command Injection | Hackazon |
| 🔴 Immediate | Hardcoded Credentials | Hackazon |
| 🔴 Immediate | Remote File Inclusion | MoinMaster |
| 🟠 High | XSS | Both |
| 🟠 High | CSRF | Both |
| 🟠 High | Weak Password Hashing (MD5) | MoinMaster |
| 🟠 High | Insecure Session Management | Hackazon |
| 🟡 Medium | IDOR | MoinMaster |
| 🟡 Medium | Insecure File Upload | Hackazon |
| 🟡 Medium | Information Disclosure | Hackazon |

---

## 📚 References

- SonarQube Documentation (2023) — https://docs.sonarqube.org/latest/
- Semgrep Documentation (2023) — https://semgrep.dev/docs/
- OWASP Top 10 — https://owasp.org/www-project-top-ten/
- CWE/SANS Top 25 — https://cwe.mitre.org/top25/

---

## 👤 Author

**Anshio Renin Micheal Antony Xavier Soosammal**  
MSc Cybersecurity | Dublin Business School  
Student No: 20036753  
🔗 [LinkedIn](www.linkedin.com/in/anshio-renin-ms) | CC ISC2 Certified | Open to Work in Ireland
