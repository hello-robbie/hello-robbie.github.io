````markdown
# 🧭 Web App Default Paths Cheat Sheet

When scanning a network or analyzing a web server, knowing the default paths of popular tools can save you a ton of time. Here’s a field-ready cheat sheet for quickly identifying web apps based on their common routes.

---

## 🧱 Node-RED

| Purpose       | Path        |
|---------------|-------------|
| Editor UI     | `/` or `/red` |
| Dashboard UI  | `/ui`       |
| Flow JSON API | `/flows`    |

---

## 🔥 Flask / Werkzeug Apps

| Purpose        | Path Examples                              |
|----------------|---------------------------------------------|
| Admin/Login    | `/login`, `/admin`                         |
| API Entry      | `/api/`, `/api/status`, `/api/gate`        |
| Flask Debug Panel | _Only if misconfigured_ — look for `/console` in stack traces |

---

## 🐘 phpMyAdmin / MySQL Admin

| Tool        | Common Paths                                 |
|-------------|-----------------------------------------------|
| phpMyAdmin  | `/phpmyadmin`, `/pma`, `/mysqladmin`, `/dbadmin` |

---

## 🧪 Jenkins

| Purpose        | Path                |
|----------------|---------------------|
| Dashboard      | `/`, `/jenkins`, `/dashboard` |
| Script Console | `/script`           |

---

## 📊 Grafana

| Purpose         | Path                      |
|------------------|---------------------------|
| Login/Dashboard  | `/login`, `/d/`, `/dashboard/` |

---

## 📈 Kibana

| Purpose   | Path         |
|-----------|--------------|
| Dashboard | `/app/kibana` |

---

## 📉 Prometheus

| Purpose  | Path             |
|----------|------------------|
| Web UI   | `/graph`, `/metrics` |

---

## 🏭 Industrial / OT Interfaces

These should never be exposed to the public internet, but if you encounter them:

| Tool            | Default Port | Path Hints                     |
|-----------------|---------------|--------------------------------|
| Web HMI         | varies        | `/`, `/ui`, `/hmi`, `/dashboard` |
| PLC Admin UI    | 80, 443       | `/plc`, `/admin`, `/setup`     |

---

## 🎁 Bonus Tip: Auto-Discovery Tools

If guessing doesn't work, try some recon tools:

```
whatweb http://<ip>:<port>
````

or

```
gobuster dir -u http://<ip>:<port> -w /usr/share/wordlists/dirb/common.txt
```

These tools can brute-force or fingerprint the server to uncover hidden or non-standard paths.

---

Stay curious, and don’t forget to double-check if what you find **should** be exposed at all.
