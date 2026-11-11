# python-selenium
cat <<'EOF' > ~/selenium-lab/validate.sh
#!/usr/bin/env bash
cd ~/selenium-lab || exit 1
source .venv/bin/activate
pass=0; fail=0
check() {
  local desc="$1"; shift
  if "$@" >/dev/null 2>&1; then echo "PASS  $desc"; pass=$((pass+1));
  else echo "FAIL  $desc"; fail=$((fail+1)); fi
}
gql() {
  curl -s -X POST -H "Content-Type: application/json" --data "{\"query\":\"$1\"}" http://localhost:4444/graphql
}
export -f gql

check "T1 webapp /healthz returns ok"        bash -c '[ "$(curl -s http://localhost:5000/healthz)" = "ok" ]'
check "T1 webapp container healthy"          bash -c 'docker compose ps webapp | grep -q "(healthy)"'
check "T2 grid ready with 2 nodes"           bash -c 'curl -s http://localhost:4444/status | jq -e ".value.ready and (.value.nodes|length)==2"'
check "T2 grid exposes 4 slots"              bash -c 'gql "{ grid { totalSlots } }" | jq -e ".data.grid.totalSlots==4"'
check "T3 home screenshot is a PNG"          bash -c 'file artifacts/home.png | grep -q "PNG image data"'
check "T3 no leaked sessions on the grid"    bash -c 'gql "{ grid { sessionCount } }" | jq -e ".data.grid.sessionCount==0"'
check "T4 login_flow.py completes all steps" bash -c 'python scripts/login_flow.py | grep -qF "[7] negative login error: Invalid username or password"'
check "T5 dynamic.py handles waits+alerts"   bash -c 'python scripts/dynamic.py | grep -qF "[5] alert accepted, empty row text: No tasks"'
check "T5 custom wait implemented"           bash -c 'python scripts/custom_wait.py | grep -q "custom wait ok: Total tasks:"'
check "T6 TasksPage has no TODOs left"       bash -c '! grep -q TODO framework/pages/tasks_page.py'
check "T6 suite passes on chrome"            bash -c 'cd framework && pytest -q -p no:cacheprovider -n 2 --browser chrome'
check "T6 suite passes on firefox"           bash -c 'cd framework && pytest -q -p no:cacheprovider -n 2 --browser firefox'
check "T7 failure screenshot captured"       bash -c 'file framework/artifacts/FAILED_chrome_test_intentional_failure.png | grep -q PNG'
check "T7 HTML report generated"             test -s framework/artifacts/report.html
check "T8 run_ci.sh is executable"           test -x run_ci.sh
for b in chrome firefox; do
  check "T8 CI report for $b exists"         test -s artifacts/ci/report-$b.html
  check "T8 CI junit for $b has 0 failures"  python3 -c "import xml.etree.ElementTree as E,sys; s=E.parse('artifacts/ci/junit-$b.xml').getroot().find('testsuite'); sys.exit(0 if int(s.get('tests'))>0 and s.get('failures')=='0' and s.get('errors')=='0' else 1)"
done
check "T8 CI artifacts owned by student"     bash -c '[ "$(stat -c %U artifacts/ci/report-chrome.html)" = "$USER" ]'

echo "--------------------------------------"
echo "Passed: $pass   Failed: $fail"
[ "$fail" -eq 0 ]
EOF
chmod +x ~/selenium-lab/validate.sh
~/selenium-lab/validate.sh
