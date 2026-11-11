# Python Selenium: Cross-Browser UI Test Automation on a Headless Selenium Grid

A hands-on DevOps lab that builds a small Flask application, deploys Selenium Grid 4 with Docker Compose, and creates a Python/pytest browser regression suite for Chrome and Firefox.

> **Estimated time:** 2.5–3 hours  
> **Target environment:** Ubuntu 22.04 Linux server or VM  
> **Main tools:** Python 3, Selenium WebDriver, pytest, Docker, Docker Compose, Selenium Grid 4

---

## 1. Lab scenario

You are part of the DevOps team at **Northwind Ops**. Developers maintain an internal task-tracking application called **TaskBoard**. Manual browser checks are slow and regressions sometimes reach production.

Your task is to build an automated regression-testing solution that:

- Runs without a desktop or physical browser on a Linux build agent.
- Tests the application in both Chrome and Firefox.
- Runs tests in parallel.
- Uses maintainable Python code and the Page Object Model (POM).
- Captures evidence when a test fails.
- Produces HTML and JUnit XML reports.
- Can be executed through a CI-style shell pipeline.

## 2. What you will build

```text
                         Linux host / CI agent
                                  |
                           run_ci.sh
                                  |
                         Docker Compose network
                                  |
             +--------------------+--------------------+
             |                    |                    |
        TaskBoard app        Selenium Hub         Test runner
        webapp:5000          :4444                 pytest
                                  |
                         Selenium Grid event bus
                            /             \
                           /               \
                    Chrome node        Firefox node
                    2 sessions          2 sessions
                           \               /
                            \             /
                         Browser tests
```

The Python test runner sends WebDriver commands to Selenium Grid. Grid assigns each session to a Chrome or Firefox node. The browser containers open TaskBoard using the Compose service name `webapp`, not `localhost`.

## 3. Learning objectives

By completing the lab, you will practise:

1. Building and containerising a Flask application.
2. Deploying and inspecting Selenium Grid 4.
3. Understanding the W3C WebDriver protocol through raw HTTP requests.
4. Creating and releasing remote browser sessions.
5. Using Selenium locator strategies: ID, name, CSS selector, XPath and tag name.
6. Synchronising tests with explicit waits.
7. Handling HTML dropdowns and JavaScript confirm dialogs.
8. Building a pytest framework with fixtures, markers and POM.
9. Capturing screenshots and page source on test failure.
10. Producing HTML and JUnit reports and running tests in parallel.
11. Packaging a test runner and executing a CI-style pipeline.
12. Diagnosing common Docker, networking and Selenium failures.

---

## 4. Prerequisites

You should be comfortable with:

- Linux shell commands, directories, permissions and environment variables.
- Basic Python: functions, classes, imports and exceptions.
- Basic HTML and CSS: elements, IDs, classes and attributes.
- Docker images, containers and port mapping.

### Recommended machine resources

| Resource | Recommendation |
|---|---|
| OS | Ubuntu 22.04 |
| CPU | 2 or more vCPUs |
| RAM | 4–8 GB |
| Free disk | At least 10 GB |
| Access | sudo and internet access |

The Selenium browser images are large. The three Grid images may consume around 3 GB or more.

---

## 5. Lab roadmap

| Task | What you will do | Approx. time |
|---|---|---:|
| 1 | Build and containerise TaskBoard | 15 min |
| 2 | Deploy Selenium Grid 4 and use raw WebDriver HTTP | 20 min |
| 3 | Create a remote session, save a screenshot and inspect session leaks | 20 min |
| 4 | Practise locator strategies and element interactions | 25 min |
| 5 | Use explicit waits, dropdowns and JavaScript alerts | 25 min |
| 6 | Build a pytest framework using the Page Object Model | 30 min |
| 7 | Capture failure evidence, generate reports and run in parallel | 20 min |
| 8 | Containerise the test runner and run the CI pipeline | 20 min |

---

# Part A — Prepare the Linux environment

## 6. Install base packages

Update the package index and install the required utilities:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg jq file \
  python3-venv python3-pip
```

Verify:

```bash
python3 --version
jq --version
```

Expected: Python 3.10 or a compatible Python 3 release, and jq installed.

## 7. Install Docker Engine and Compose

The lab uses Docker's official Ubuntu apt repository.

```bash
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update

sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

Allow your user to run Docker:

```bash
sudo usermod -aG docker "$USER"
```

**Important:** Log out and reconnect (or start a new login session) so the group membership takes effect. Then verify:

```bash
docker run --rm hello-world
docker compose version
```

If you need to test Docker before reconnecting, use `sudo docker run --rm hello-world`.

## 8. Create the project structure and Python environment

```bash
mkdir -p ~/selenium-lab/{app,scripts,solutions,artifacts,framework/pages,framework/tests}
cd ~/selenium-lab

python3 -m venv .venv
source .venv/bin/activate
```

Create `framework/requirements.txt`:

```text
selenium==4.18.1
pytest==8.0.2
pytest-html==4.1.1
pytest-xdist==3.5.0
```

Install the Python dependencies:

```bash
pip install --upgrade pip
pip install -r framework/requirements.txt

python -c "import selenium; print('selenium', selenium.__version__)"
```

Whenever you reconnect to the Linux host, activate the environment again:

```bash
cd ~/selenium-lab
source .venv/bin/activate
```

## 9. Download the Selenium Grid images

The lab pins the Grid image tags to keep the hub and browser nodes aligned.

```bash
docker pull selenium/hub:4.18.1-20240224
docker pull selenium/node-chrome:4.18.1-20240224
docker pull selenium/node-firefox:4.18.1-20240224

docker images | grep selenium
```

---

# Part B — Build the TaskBoard application

## 10. Task 1: Build and containerise the application

TaskBoard is the application under test. It includes:

- A home page and login form.
- A task table with default tasks.
- A task-creation form and priority dropdown.
- A delayed statistics widget that loads data through AJAX/fetch.
- A JavaScript confirmation dialog for deleting tasks.
- A health endpoint for readiness checks.

### 10.1 Create the Flask application

Create `app/app.py` and use the Flask application source supplied in the lab handout. The application uses these demo credentials:

| Field | Value |
|---|---|
| Username | `admin` |
| Password | `s3cret` |

The initial task list is:

- Patch web servers — high priority
- Rotate log files — low priority

The app stores tasks in the signed session cookie, isolating browser sessions from one another for parallel testing.

### 10.2 Create the application Dockerfile

Create `app/Dockerfile`:

```dockerfile
FROM python:3.10-slim
WORKDIR /app
RUN pip install --no-cache-dir flask==3.0.2 werkzeug==3.0.1 gunicorn==21.2.0
COPY app.py .
EXPOSE 5000
CMD ["gunicorn", "-w", "2", "-b", "0.0.0.0:5000", "app:app"]
```

### 10.3 Start the application with Compose

Create the initial `docker-compose.yml`:

```yaml
services:
  webapp:
    build: ./app
    image: taskboard:lab
    ports:
      - "5000:5000"
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:5000/healthz')"]
      interval: 5s
      timeout: 3s
      retries: 10
```

Build and start:

```bash
cd ~/selenium-lab
docker compose up -d --build webapp
docker compose ps
```

Verify the application:

```bash
curl -s http://localhost:5000/healthz; echo
curl -s http://localhost:5000/ | grep -E "<title>|<h1>"
curl -s http://localhost:5000/login | grep -E "<input|<button"
```

Expected health response:

```text
ok
```

Verify login using curl:

```bash
curl -s -c /tmp/jar -o /dev/null -w '%{http_code} %{redirect_url}\n' \
  -d username=admin -d password=s3cret http://localhost:5000/login

curl -s -b /tmp/jar http://localhost:5000/api/stats
```

Expected: login redirects to `/tasks` with HTTP 302; the initial statistics should report two tasks and one high-priority task.

**Task 1 checkpoint:** TaskBoard is running and its `/healthz` endpoint returns `ok`.

---

# Part C — Deploy Selenium Grid 4

## 11. Task 2: Add the hub and browser nodes

Replace the initial Compose file with the full Grid configuration from the lab handout. It must contain four services:

- `webapp`
- `selenium-hub`
- `chrome`
- `firefox`

Use these pinned images:

```text
selenium/hub:4.18.1-20240224
selenium/node-chrome:4.18.1-20240224
selenium/node-firefox:4.18.1-20240224
```

Each browser node is configured for two concurrent sessions. Keep `shm_size: "1gb"` on both browser nodes.

Start the stack:

```bash
cd ~/selenium-lab
docker compose up -d
docker compose ps
```

### 11.1 Understand the important Grid settings

| Setting | Purpose |
|---|---|
| `SE_EVENT_BUS_HOST` | Hostname of the Selenium hub |
| `SE_EVENT_BUS_PUBLISH_PORT` | Grid event-bus publish port |
| `SE_EVENT_BUS_SUBSCRIBE_PORT` | Grid event-bus subscribe port |
| `SE_NODE_MAX_SESSIONS` | Maximum sessions configured for a node |
| `SE_NODE_OVERRIDE_MAX_SESSIONS` | Allows the configured session limit |
| `shm_size` | Shared memory available to browser containers |

The event-bus ports are used inside the Compose network. They do not need to be published to the host for this lab.

### 11.2 Wait for Grid readiness

```bash
for i in $(seq 1 30); do
  if curl -sf http://localhost:4444/status | jq -e '.value.ready' >/dev/null; then
    echo "Grid ready"
    break
  fi
  sleep 2
done
```

Inspect status and nodes:

```bash
curl -s http://localhost:4444/status | jq -r \
  '.value.ready, .value.message, (.value.nodes | length)'

curl -s http://localhost:4444/status | jq -r \
  '.value.nodes[] | "\(.slots[0].stereotype.browserName) slots=\(.slots | length) \(.availability)"'
```

Expected: Grid ready, two nodes, Chrome and Firefox nodes registered.

### 11.3 Query the GraphQL endpoint

```bash
curl -s -X POST -H "Content-Type: application/json" \
  --data '{"query":"{ grid { totalSlots nodeCount maxSession sessionCount } }"}' \
  http://localhost:4444/graphql | jq
```

You should see four total slots when both nodes have two slots configured.

### 11.4 Practise the raw W3C WebDriver protocol

Create a browser session using HTTP:

```bash
SID=$(curl -s -X POST http://localhost:4444/session \
  -H 'Content-Type: application/json' \
  -d '{"capabilities":{"alwaysMatch":{"browserName":"chrome"}}}' \
  | jq -r '.value.sessionId')

echo "Session: $SID"
```

Navigate, inspect the title, locate an element and close the session:

```bash
curl -s -X POST http://localhost:4444/session/$SID/url \
  -H 'Content-Type: application/json' \
  -d '{"url":"http://webapp:5000/"}'

curl -s http://localhost:4444/session/$SID/title

curl -s -X POST http://localhost:4444/session/$SID/element \
  -H 'Content-Type: application/json' \
  -d '{"using":"css selector","value":"h1"}'

curl -s -X DELETE http://localhost:4444/session/$SID
```

**Task 2 checkpoint:** The Grid status endpoint reports ready, both browser nodes are registered, and a raw WebDriver session can be created and deleted.

---

# Part D — Practise Selenium WebDriver

## 12. Task 3: Remote session and screenshot

Create `scripts/first_session.py` using the example in the lab handout. It should:

1. Create Chrome options.
2. Connect to `http://localhost:4444` using `webdriver.Remote`.
3. Print the session ID and browser capabilities.
4. Open `http://webapp:5000/`.
5. Print the page title and URL.
6. Save a screenshot to `artifacts/home.png`.
7. Always call `driver.quit()` in a `finally` block.

Run it:

```bash
cd ~/selenium-lab
python scripts/first_session.py
file artifacts/home.png
```

### Important: localhost inside a container

The Python script runs on the Linux host, but the browser runs inside a browser container. When the browser opens `localhost`, it refers to the browser container itself.

Use:

```text
http://webapp:5000
```

Do not use `http://localhost:5000` as the URL opened by a Grid browser.

### Practise a leaked session

The handout also provides `scripts/leak.py`, which deliberately omits `driver.quit()`. Run it, inspect `sessionCount` through GraphQL, find the session ID in `/status`, and delete the session using:

```bash
curl -s -X DELETE http://localhost:4444/session/SESSION_ID
```

Replace `SESSION_ID` with the actual ID returned by the Grid.

**Task 3 checkpoint:** You have a screenshot and can explain why `driver.quit()` is essential.

## 13. Task 4: Locator strategies and interactions

Create `scripts/login_flow.py` from the starter code in the handout and complete its TODOs.

You will practise:

| Locator | Example target |
|---|---|
| `By.ID` | Username field |
| `By.NAME` | Password field |
| `By.CSS_SELECTOR` | Login submit button |
| `By.TAG_NAME` | Page heading |
| `By.XPATH` | Task table rows |
| Relative element lookup | Cells inside a selected row |

The script should log in, print the heading and current user, read the initial task rows, log out, and verify that a wrong password produces the expected error message.

Run:

```bash
python scripts/login_flow.py
```

Expected key results:

```text
[3] heading: My Tasks
[4] logged in as: admin
[5] task rows: 2
[6] first task: Patch web servers (high)
[7] negative login error: Invalid username or password
```

Use the reference implementation in the handout only after attempting the TODOs yourself.

**Task 4 checkpoint:** All seven output steps are present and no value is `None`.

## 14. Task 5: Explicit waits, dropdowns and alerts

Create `scripts/dynamic.py` from the handout.

This exercise covers three common browser-automation problems:

1. Selecting a task priority using Selenium's `Select` class.
2. Waiting for the delayed statistics widget to finish loading.
3. Handling a JavaScript `confirm()` dialog with `accept()` and `dismiss()`.

Run:

```bash
python scripts/dynamic.py
```

The statistics widget takes about 2.5 seconds to return. A lookup immediately after clicking the button may fail because the result is not yet present.

Use `WebDriverWait` and an expected condition rather than a fixed sleep:

```python
WebDriverWait(driver, 10).until(
    EC.visibility_of_element_located((By.ID, "stats-total"))
)
```

The handout also includes `scripts/custom_wait.py`. Complete its TODO with a custom callable that returns true when the statistics text begins with `Total tasks:`.

**Important:** Use explicit waits consistently. Do not mix implicit and explicit waits, and do not use `time.sleep()` as the test synchronisation strategy.

**Task 5 checkpoint:** The script adds a task, waits for statistics, dismisses a confirmation dialog, and accepts a confirmation dialog.

---

# Part E — Build a maintainable pytest framework

## 15. Task 6: Page Object Model and pytest

Work inside:

```bash
cd ~/selenium-lab/framework
```

The framework separates browser setup, page interactions and test assertions.

### 15.1 Project structure

The completed project should look similar to:

```text
selenium-lab/
├── app/
│   ├── app.py
│   └── Dockerfile
├── scripts/
│   ├── first_session.py
│   ├── leak.py
│   ├── login_flow.py
│   ├── dynamic.py
│   └── custom_wait.py
├── solutions/
├── framework/
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── requirements.txt
│   ├── pytest.ini
│   ├── conftest.py
│   ├── pages/
│   │   ├── __init__.py
│   │   ├── base_page.py
│   │   ├── login_page.py
│   │   └── tasks_page.py
│   └── tests/
│       ├── test_login.py
│       └── test_tasks.py
├── artifacts/
├── docker-compose.yml
├── run_ci.sh
└── validate.sh
```

### 15.2 Create pytest configuration and fixtures

Create `framework/pytest.ini` and `framework/conftest.py` using the complete examples in the handout.

The fixtures and command-line options should support:

- `--browser chrome` or `--browser firefox`
- `--grid-url`
- `--app-url`
- A function-scoped remote WebDriver fixture.
- `driver.quit()` after each test.

### 15.3 Create the Page Objects

Create these files using the handout:

- `pages/base_page.py` — common navigation, find, click and type methods.
- `pages/login_page.py` — login form locators and login actions.
- `pages/tasks_page.py` — task table, task form, statistics and delete-dialog actions.

Complete all six TODOs in `TasksPage`:

1. Enter a task title.
2. Select a priority by visible text.
3. Submit the task.
4. Wait for the new task row.
5. Load statistics and return integer values.
6. Handle the confirmation alert and wait for the empty state when accepted.

The expected method behaviours are:

| Method | Expected behaviour |
|---|---|
| `add_task(title, priority_label)` | Add a task and wait for its row |
| `load_stats()` | Return a dictionary such as `{"total": 3, "high": 2}` |
| `clear_all(accept=True)` | Return the alert text; accept or dismiss the dialog |

### 15.4 Create the test files

Create `tests/test_login.py` and `tests/test_tasks.py` using the test cases in the handout.

The suite covers:

- Home page title.
- Valid login.
- Invalid login with multiple data combinations.
- Default tasks.
- Adding a task.
- Empty task title validation.
- Delayed statistics.
- Cancelling task deletion.
- Confirming task deletion.
- An intentional failure used only to demonstrate failure artifacts.

Run Chrome tests:

```bash
pytest --browser chrome -v
```

Run Firefox tests:

```bash
pytest --browser firefox -q
```

Run only smoke tests:

```bash
pytest --browser chrome -m smoke -q
```

The intentional failure test is skipped by default. The normal suite should report 11 passed and 1 skipped when all supplied tests are present and working.

**Task 6 checkpoint:** The suite passes in Chrome and Firefox, and the `TasksPage` TODOs are complete.

---

## 16. Task 7: Failure evidence, reports and parallel execution

A test failure should leave enough evidence to diagnose what happened.

Replace `framework/conftest.py` with the extended version in the handout. It adds a `pytest_runtest_makereport` hook that captures:

- A screenshot (`.png`).
- The page source (`.html`).
- The current page URL in the HTML report.
- An embedded screenshot in the pytest-html report.

### 16.1 Trigger the controlled failure

From `framework/`, run:

```bash
FAIL_DEMO=1 pytest --browser chrome \
  --html=artifacts/report.html --self-contained-html \
  --junitxml=artifacts/junit.xml -q
```

The intentional test should fail because it expects `My Projects` while the application heading is `My Tasks`. This is deliberate: it proves the failure hook works.

Inspect the files:

```bash
ls -1 artifacts/
file artifacts/FAILED_chrome_test_intentional_failure.png
grep -o "<h1>[^<]*</h1>" artifacts/FAILED_chrome_test_intentional_failure.html
```

### 16.2 Understand the reports

| Artifact | Purpose |
|---|---|
| `report.html` | Human-readable test report |
| `junit.xml` | Machine-readable test results for CI systems |
| `FAILED_*.png` | Screenshot of the failing browser state |
| `FAILED_*.html` | Page source captured at failure time |

### 16.3 Run tests in parallel

Compare serial and parallel execution:

```bash
pytest --browser chrome -q
pytest --browser chrome -q -n 2
```

Parallel execution works because each test gets its own browser session, TaskBoard keeps state in each session cookie, and generated task names use unique values.

Do not request more concurrent sessions than your machine can reasonably support. The Grid queues session requests when all slots are occupied.

**Task 7 checkpoint:** You can deliberately create a failure, find its screenshot and DOM file, and generate HTML/JUnit reports.

---

# Part F — Containerise the runner and create a CI pipeline

## 17. Task 8: Build the test-runner image

Create `framework/.dockerignore`:

```text
artifacts/
__pycache__/
**/__pycache__/
.pytest_cache/
```

Create `framework/Dockerfile`:

```dockerfile
FROM python:3.10-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /tests
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
ENTRYPOINT ["pytest", "-p", "no:cacheprovider"]
```

## 18. Update Docker Compose

Replace the root `docker-compose.yml` with the final configuration from the handout. Keep the four existing services and add a `tests` service.

Important settings for the test runner:

- `profiles: ["tests"]` prevents the test runner from starting during a normal `docker compose up`.
- `GRID_URL` must be `http://selenium-hub:4444` inside the Compose network.
- `APP_URL` must be `http://webapp:5000` inside the Compose network.
- Mount `./artifacts/ci` into `/tests/artifacts`.
- Set the runner user from `HOST_UID` and `HOST_GID` so generated files belong to your Linux user.

## 19. Create the CI pipeline script

Create `run_ci.sh` at the project root using the complete script in the handout.

The pipeline performs these stages:

```text
Start application and Grid
          |
Build test-runner image
          |
Wait for Grid readiness
          |
Run Chrome smoke tests
          |
   Smoke gate passed?
       /       \
     No         Yes
     |           |
  Exit 1     Full Chrome suite
                 |
           Full Firefox suite
                 |
          Return test exit code
```

Make the script executable and run it:

```bash
cd ~/selenium-lab
chmod +x run_ci.sh
./run_ci.sh
```

Expected key output:

```text
==> Starting stack
==> Waiting for Selenium Grid (2 nodes)
Grid ready
==> Smoke gate (chrome)
==> Full suite on chrome
==> Full suite on firefox
==> Pipeline finished with exit code 0
```

### 19.1 Prove that the pipeline fails correctly

Run the intentional failure inside the test container:

```bash
HOST_UID=$(id -u) HOST_GID=$(id -g) \
docker compose run --rm -e FAIL_DEMO=1 -e BROWSER=firefox \
tests -q -k intentional_failure

echo "exit code: $?"
```

The command should return a non-zero exit code and create failure artifacts. A CI pipeline must not report success when tests fail.

After the demonstration, run the normal pipeline again:

```bash
./run_ci.sh
```

**Task 8 checkpoint:** The pipeline starts the stack, waits for Grid readiness, runs the smoke gate and both browser suites, creates reports and returns a meaningful exit code.

---

# Part G — Challenge extension

## 20. Optional advanced exercise: Data-driven regression

Complete the challenge from the lab handout after the core lab is working.

### Requirements

1. Create `framework/data/login_cases.csv` with columns `username,password,expected`.
2. Include at least six cases: valid credentials, wrong password, unknown user, empty fields, case sensitivity (`Admin`) and whitespace around the username.
3. Create `framework/tests/test_login_data.py`.
4. Load the CSV at collection time and use `pytest.mark.parametrize` with readable test IDs.
5. Create `framework/pages/conditions.py` with a reusable `row_count_is(locator, n)` condition. It should return the matching elements when exactly `n` are found.
6. Add a test that creates three tasks with different priorities and waits for exactly five rows.
7. Extend `run_ci.sh` to accept a `BROWSERS` environment variable, defaulting to `chrome firefox`, and print a per-browser summary from JUnit XML.

### Acceptance tests

```bash
cd ~/selenium-lab/framework
pytest --browser chrome -k login_data -v
```

Then try:

```bash
cd ~/selenium-lab
BROWSERS=firefox ./run_ci.sh
```

Editing one expected value in the CSV to an incorrect value should make the corresponding test fail and produce a screenshot.

---

## 21. Final validation

Create `validate.sh` at the project root using the validation script in the handout. It checks:

- TaskBoard health and container health.
- Selenium Grid readiness and four available slots.
- Screenshot creation and absence of leaked sessions.
- Stand-alone login and wait scripts.
- Completion of the Page Object TODOs.
- Chrome and Firefox pytest runs.
- Failure screenshot and HTML report generation.
- CI script permissions and report generation.
- CI artifact ownership.

Run:

```bash
cd ~/selenium-lab
chmod +x validate.sh
./validate.sh
```

The handout's expected result is:

```text
--------------------------------------
Passed: 20   Failed: 0
```

If a check fails, use the troubleshooting section below and rerun the relevant task before running the full validation again.

---

## 22. Troubleshooting guide

| Problem | Likely cause | What to check |
|---|---|---|
| APT lock permission denied | Command ran without root privileges | Use `sudo apt-get update` |
| Docker socket permission denied | Docker group membership is not active | Log out and reconnect; check `id -nG` |
| Grid connection refused | Hub is stopped or still starting | `docker compose ps` and `docker compose logs selenium-hub` |
| Grid ready is false or no nodes appear | Node cannot reach the event bus | Check `SE_EVENT_BUS_*` values and node logs |
| Browser gets connection refused | Browser URL uses host `localhost` | Use `http://webapp:5000` |
| Session cannot be created | Matching browser node is unavailable or crashed | Inspect `/status` node stereotypes and logs |
| Tests wait for several minutes | Grid slots may be held by leaked sessions | Ensure `driver.quit()` runs; inspect and delete leaked sessions |
| Browser or tab crashes | Shared memory is too small | Keep `shm_size: "1gb"` or increase it |
| `NoSuchElementException` | Locator is wrong or element is not present yet | Inspect the live DOM and use an appropriate wait |
| `TimeoutException` | Wait condition did not become true | Check the condition and realistic timeout |
| `UnexpectedAlertPresentException` | A JavaScript dialog is still open | Wait for `EC.alert_is_present()` and accept/dismiss it |
| `StaleElementReferenceException` | Page changed and an old element was reused | Locate the element again after navigation |
| `ModuleNotFoundError: pages` | pytest started from the wrong directory or package init is missing | Run from `framework/`; check `pytest.ini` and `pages/__init__.py` |
| CI artifacts owned by root | UID/GID were not passed to Compose | Use `run_ci.sh` or export `HOST_UID` and `HOST_GID` |
| HTML report lacks screenshot | pytest-html is missing or failure occurred during setup | Check `pip show pytest-html`; hook captures call-phase failures |

Useful diagnostic commands:

```bash
docker compose ps
docker compose logs --tail=100 selenium-hub chrome firefox
curl -s http://localhost:4444/status | jq
curl -s http://localhost:4444/status | jq \
  '[.value.nodes[].slots[] | select(.session != null)] | length'
```

---

## 23. Cleanup

Stop and remove the Compose containers, network and locally built images:

```bash
cd ~/selenium-lab
docker compose --profile tests down --rmi local --remove-orphans
docker compose ps -a
```

Optionally remove the downloaded Selenium images to reclaim disk space:

```bash
docker image rm \
  selenium/hub:4.18.1-20240224 \
  selenium/node-chrome:4.18.1-20240224 \
  selenium/node-firefox:4.18.1-20240224

docker image prune -f
```

Optionally delete the project and virtual environment:

```bash
deactivate 2>/dev/null || true
rm -rf ~/selenium-lab
```

Only run the final command if you no longer need your lab files or reports.

---

## 24. Key takeaways

- **Selenium WebDriver is a protocol client.** Python commands are translated into W3C WebDriver HTTP requests.
- **Selenium Grid enables remote browser execution.** A hub routes sessions to browser nodes, making cross-browser and parallel testing possible.
- **Container networking matters.** A browser inside Docker must use service DNS names such as `webapp`, not the host's `localhost`.
- **Reliable tests need explicit synchronisation.** Prefer `WebDriverWait` and expected conditions over fixed sleeps.
- **Page Object Model improves maintainability.** Locators and page actions stay separate from test assertions.
- **Every browser session must be released.** Use `try/finally` or pytest fixtures that always call `driver.quit()`.
- **Failure evidence speeds up debugging.** Screenshots, page source, HTML reports and JUnit XML make test results useful to both engineers and CI systems.
- **Parallel testing requires isolation.** Each test needs its own browser session and independent test data.
- **A CI pipeline must report failures honestly.** Readiness checks, smoke gates and correct exit codes help prevent broken releases.

## 25. Suggested GitHub repository checklist

Before publishing your lab, make sure your repository contains:

- [ ] `README.md`
- [ ] `app/` with Flask source and Dockerfile
- [ ] `scripts/` with stand-alone Selenium exercises
- [ ] `framework/` with requirements, pytest configuration, page objects and tests
- [ ] Root `docker-compose.yml`
- [ ] `run_ci.sh`
- [ ] `validate.sh`
- [ ] `.gitignore` excluding `.venv/`, `__pycache__/`, `.pytest_cache/` and generated artifacts where appropriate
- [ ] Screenshots or report examples only if they do not expose secrets or unnecessary machine-specific information

---

**Lab completion goal:** You should be able to start the TaskBoard application and Selenium Grid, run the same regression tests in Chrome and Firefox, inspect evidence for failures, and execute the workflow through a repeatable CI-style script.
