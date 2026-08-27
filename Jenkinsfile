pipeline {
    agent any
    environment {
        SWV_BACKEND_URL = 'http://codevi-backend:13000/api'
        TREESITTER_PARSER_URL = 'http://codevi-parser-treesitter:3001/analyze'
        EXAMINE_URL = 'http://codevi-pyexamine:3003/codegraph'
        REPO_OWNER = 'jej031020'
        REPO_NAME = 'AI-Security-Gateway'
    }
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Analyze & Report Snapshot') {
            steps {
                sh '''
set -e
git config --global --add safe.directory "*"

echo ">>> Zipping source (python) ..."
python3 - <<'ZP'
import zipfile, os
skip = {".git", "node_modules", "dist", "__pycache__", ".venv", "venv"}
z = zipfile.ZipFile("code_package.zip", "w", zipfile.ZIP_DEFLATED)
for root, dirs, files in os.walk("."):
    dirs[:] = [d for d in dirs if d not in skip]
    for f in files:
        p = os.path.join(root, f)
        if p.endswith("code_package.zip"):
            continue
        try:
            if os.path.isfile(p):  # 깨진 심볼릭 링크/특수파일 건너뜀
                z.write(p)
        except OSError:
            pass
z.close()
ZP

echo ">>> Sending to tree-sitter parser ..."
curl -s -X POST "$TREESITTER_PARSER_URL" -F "file=@code_package.zip" -o ast.json

echo ">>> Sending to examine (pyexamine) /codegraph ..."
curl -s -X POST "$EXAMINE_URL" -F "file=@code_package.zip" -o codegraph.json || echo '{}' > codegraph.json

echo ">>> Building build-report payload ..."
python3 - <<PY
import json, os
d = json.load(open("ast.json"))
nodes = (d.get("data") or d).get("nodes") if isinstance(d, dict) else None
if nodes is None:
    nodes = []

# examine 캐노니컬 코드그래프(있으면): units[].metrics 에서 클래스별 CK 파생
# examine metrics 형태: {"type":"ck","version":"1","values":{cbo,wmc,lcom,...}}
# → 프론트/AstClass.metrics 는 평면 {cbo,wmc,lcom,...} 을 기대하므로 values 를 펴고 null→0.
cg = None
class_metrics = []
try:
    cg = json.load(open("codegraph.json"))
    for u in (cg.get("units") or []):
        kind = str(u.get("kind", "")).lower()
        m = u.get("metrics")
        if m and ("class" in kind or "interface" in kind or "struct" in kind):
            vals = m.get("values") if isinstance(m, dict) and isinstance(m.get("values"), dict) else m
            flat = {k: (0 if v is None else v) for k, v in (vals or {}).items()}
            if flat:
                class_metrics.append({
                    "file": u.get("file") or "",
                    "name": u.get("name") or "",
                    "metrics": flat,
                })
except Exception:
    cg = None

proj = {
    "teamName": "$REPO_NAME",
    "jenkinsJobName": os.environ["JOB_NAME"],
    "sonarProjectKey": os.environ["JOB_NAME"],
    "analysis": {
        "jobName": os.environ["JOB_NAME"],
        "buildNumber": int(os.environ.get("BUILD_NUMBER", "0")),
        "status": "SUCCESS",
        "buildUrl": os.environ.get("BUILD_URL", "UNKNOWN"),
        "commitHash": os.environ.get("GIT_COMMIT", "unknown"),
    },
    "astNodes": nodes,
    "classMetrics": class_metrics,
}
if cg and cg.get("units"):
    proj["codeGraph"] = cg
payload = {"username": ["$REPO_OWNER"], "ProjectDto": proj}
json.dump(payload, open("payload.json", "w"))
print(">>> nodes:", len(nodes), "classMetrics:", len(class_metrics))
PY

echo ">>> Posting snapshot to backend ..."
CODE=$(curl -s -o resp.txt -w "%{http_code}" -X POST "$SWV_BACKEND_URL/users/build-report" -H "Content-Type: application/json" -d @payload.json)
echo ">>> build-report HTTP $CODE"
head -c 400 resp.txt || true
echo ""
[ "$CODE" = "200" ] || [ "$CODE" = "201" ] || { echo ">>> Snapshot POST failed"; exit 1; }
echo ">>> Snapshot created."
'''
            }
        }
    }
    post {
        always { sh 'rm -f code_package.zip ast.json codegraph.json payload.json resp.txt || true' }
    }
}