# CampusFix

**A campus complaint platform with AI-based categorisation and priority scoring.**

Students report campus problems (Wi-Fi down, broken projector, AC not working, lab equipment faults). An ML model sorts each complaint into a category, a priority engine scores how urgent it is, and staff see everything in one list, most urgent first.

![Python](https://img.shields.io/badge/Python-3.13-blue)
![FastAPI](https://img.shields.io/badge/FastAPI-0.115-009688)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-336791)
![scikit--learn](https://img.shields.io/badge/scikit--learn-ML-F7931E)

---

## Features

| For students | For staff |
| --- | --- |
| Log in with email + password, Google, or GitHub | See every complaint from every student |
| Raise a complaint with title, location and description | List is sorted by priority (Critical → Low), then newest |
| Answer 4 quick questions that drive the priority | Update status: Open → In Progress → Resolved |
| Track the status of your own complaints | Override the AI-suggested priority |

**AI parts**

- **Complaint classifier** (`ai/classifier.py`) – TF-IDF + Logistic Regression that reads the complaint text and predicts one of **IT, Classroom, Maintenance, Lab Equipment**. Trained on 200 labelled complaints (`ai/data/complaints.csv`).
- **Priority engine** (`ai/priority.py`) – a transparent rule-based score (4–13) from four inputs, mapped to **Critical / High / Medium / Low**.

**Security**

- Passwords hashed with bcrypt; never stored in plain text
- Login kept in a signed, HttpOnly session cookie
- 5 wrong passwords in 5 minutes → login blocked for that email
- Role checks on every API (students can't change status, staff can't post complaints)
- Google tokens verified against our client ID; GitHub OAuth uses a random `state` to block forged callbacks
- All SQL is parameterised

---

## How the priority is calculated

```
score = students affected + severity + academic impact + duration
```

| Input | 1 point | 2 points | 3 points | 4 points |
| --- | --- | --- | --- | --- |
| Students affected | < 10 | 10–49 | 50–99 | 100+ |
| Severity | minor | moderate | severe | – |
| Academic impact | low | medium | high | – |
| Duration | < 8 h | 8–23 h | 24 h+ | – |

| Score | Priority |
| --- | --- |
| 11–13 | 🔴 Critical |
| 8–10 | 🟠 High |
| 5–7 | 🟡 Medium |
| 4 | 🟢 Low |

Example: 100 students, severe, high academic impact, 12 hours → 4 + 3 + 3 + 2 = **12 → Critical**.

---

## Classifier accuracy

From `python -m ai.evaluate` (200 complaints, ~50 per category):

| Metric | Result |
| --- | --- |
| 5-fold cross-validation | **94.5 %** (± 2.9 %) |
| Test accuracy (80/20 split) | **97.6 %** |

---

## Tech stack

| Layer | Tech |
| --- | --- |
| Frontend | HTML, CSS, vanilla JavaScript |
| Backend | Python, FastAPI, Uvicorn |
| Database | PostgreSQL (psycopg 3) |
| ML | scikit-learn, joblib |
| Auth | bcrypt, Starlette sessions, Google Sign-In, GitHub OAuth |

---

## Project structure

```
campusfix-devz/
├── ai/
│   ├── classifier.py      # train + load the category model
│   ├── evaluate.py        # accuracy report for the classifier
│   ├── priority.py        # rule-based priority engine
│   ├── model.pkl          # trained model
│   └── data/complaints.csv
├── backend/
│   ├── main.py            # FastAPI app, serves the frontend too
│   ├── auth.py            # login, Google/GitHub sign-in, logout
│   ├── complaints.py      # complaint APIs
│   └── db.py              # PostgreSQL helper
├── database/
│   └── database.sql       # tables + demo users
├── frontend/              # index.html (login), home.html (dashboard), JS, CSS
├── requirements.txt
└── secrets.example.bat    # template for OAuth keys
```

---

## Getting started

### Prerequisites

- Python 3.11+
- PostgreSQL 14+

### 1. Clone and install

```bash
git clone https://github.com/shivamkumar2220/campusfix-devz.git
cd campusfix-devz
python -m venv venv
venv\Scripts\activate            # Windows
# source venv/bin/activate       # macOS / Linux
pip install -r requirements.txt
```

### 2. Create the database

```bash
createdb -U postgres campusfix2
psql -U postgres -d campusfix2 -f database/database.sql
```

> ⚠️ `database.sql` **drops** the existing `users` and `complaints` tables. Only run it on a fresh or test database.

### 3. Configure (optional)

| Variable | Default | Purpose |
| --- | --- | --- |
| `DB_URL` | `postgresql://postgres:postgres@localhost:5432/campusfix2` | PostgreSQL connection |
| `SECRET_KEY` | `dev-only-secret` | Signs the session cookie – **change in production** |
| `GOOGLE_CLIENT_ID` | empty | Shows the Google button when set |
| `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET` | empty | Shows the GitHub button when set |

For Google/GitHub login, copy `secrets.example.bat` to `secrets.bat`, fill in the keys and run it before starting the server. For GitHub, set the OAuth app's callback URL to `http://localhost:8002/api/github-callback`.

### 4. Run

```bash
python -m uvicorn backend.main:app --port 8002
```

Open **http://localhost:8002**.

### Demo accounts

Password for all: `Student@123`

| Role | Email |
| --- | --- |
| Student | `student@campusfix.edu` |
| Staff | `anik@campusfix.edu` |

---

## API

| Method | Endpoint | Who | Description |
| --- | --- | --- | --- |
| POST | `/api/login` | anyone | Email + password login |
| GET | `/api/login-options` | anyone | Which social logins are enabled |
| POST | `/api/google-login` | anyone | Google Sign-In |
| GET | `/api/github-login` | anyone | Start GitHub OAuth |
| GET | `/api/github-callback` | GitHub | Finish GitHub OAuth |
| POST | `/api/logout` | logged in | Log out |
| GET | `/api/me` | logged in | Current user |
| GET | `/api/complaints` | logged in | Own complaints (student) or all (staff) |
| POST | `/api/complaints` | student | New complaint → returns category, score, priority |
| POST | `/api/complaints/{id}/status` | staff | Set Open / In Progress / Resolved |
| POST | `/api/complaints/{id}/priority` | staff | Override priority |

Interactive docs: **http://localhost:8002/docs**

---

## Retraining the classifier

1. Add rows to `ai/data/complaints.csv` (`complaint,category`)
2. Retrain: `python -m ai.classifier`
3. Check accuracy: `python -m ai.evaluate`

---

## Roadmap

- [ ] Image upload with complaints
- [ ] Email notifications on status change
- [ ] Analytics dashboard for staff
- [ ] Deploy with HTTPS (`https_only=True` on the session cookie)

---

## Team

Built by Team **DevZ** at Galgotias University.
