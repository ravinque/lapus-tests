pipeline {
  agent any

  options {
    buildDiscarder(logRotator(numToKeepStr: '30'))
    timeout(time: 90, unit: 'MINUTES')
    timestamps()
  }

  environment {
    LAPUS_BROWSERS           = 'chromium'
    CI                       = 'true'
    JENKINS_NODE_VERSION     = '22.14.0'
    CODEUP_REPO_URL          = 'https://codeup.aliyun.com/6305f229e8ec152a05d73b41/lapus-tests.git'
    LAPUS_CACHE_ROOT         = "${JENKINS_HOME}/.cache/lapus-automation"
    PLAYWRIGHT_BROWSERS_PATH = "${JENKINS_HOME}/.cache/lapus-automation/playwright-browsers"
    npm_config_cache         = "${JENKINS_HOME}/.cache/lapus-automation/npm"
    // PLAYWRIGHT_DOWNLOAD_HOST = 'https://npmmirror.com/mirrors/playwright'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout([
          $class           : 'GitSCM',
          branches         : [[name: '*/master']],
          doGenerateSubmoduleConfigurations: false,
          extensions       : [
            [$class: 'CleanBeforeCheckout'],
            [$class: 'CloneOption', depth: 1, shallow: true, noTags: true, honorRefspec: true]
          ],
          userRemoteConfigs: [[
            url          : "${CODEUP_REPO_URL}",
            credentialsId: 'lapus-automation'
          ]]
        ])
      }
    }

    stage('Install & E2E tests-htmlpage') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail

CACHE_ROOT="${LAPUS_CACHE_ROOT}"
NODE_VERSION="${JENKINS_NODE_VERSION}"
NODE_DIR="${CACHE_ROOT}/node-v${NODE_VERSION}-linux-x64"
NM_CACHE_DIR="${CACHE_ROOT}/node_modules"
PW_CACHE="${PLAYWRIGHT_BROWSERS_PATH}"

mkdir -p "${CACHE_ROOT}" "${PW_CACHE}" "${npm_config_cache}"
export PLAYWRIGHT_BROWSERS_PATH="${PW_CACHE}"
export npm_config_cache="${npm_config_cache}"

echo "=== Lapus CI caches (under JENKINS_HOME, survive workspace wipe) ==="
echo "CACHE_ROOT=${CACHE_ROOT}"
du -sh "${CACHE_ROOT}" 2>/dev/null || true

# Node.js — once per version
if [ ! -x "${NODE_DIR}/bin/node" ]; then
  echo "Cache miss: Node ${NODE_VERSION}"
  NODE_TAR="${CACHE_ROOT}/node-v${NODE_VERSION}-linux-x64.tar.gz"
  [ -f "${NODE_TAR}" ] || curl -fsSL "https://nodejs.org/dist/v${NODE_VERSION}/node-v${NODE_VERSION}-linux-x64.tar.gz" -o "${NODE_TAR}"
  rm -rf "${NODE_DIR}"
  tar -xzf "${NODE_TAR}" -C "${CACHE_ROOT}"
else
  echo "Cache hit: Node ${NODE_VERSION}"
fi
export PATH="${NODE_DIR}/bin:${PATH}"
node -v
npm -v

# node_modules — when package-lock.json unchanged
LOCK_HASH=$(sha256sum package-lock.json | awk '{print $1}')
LOCK_STAMP="${NM_CACHE_DIR}/.${LOCK_HASH}"
if [ -f "${LOCK_STAMP}" ] && [ -d "${NM_CACHE_DIR}/node_modules" ]; then
  echo "Cache hit: node_modules (${LOCK_HASH:0:12})"
  rm -rf node_modules
  cp -a "${NM_CACHE_DIR}/node_modules" node_modules
  npm ci --prefer-offline --no-audit --fund=false
else
  echo "Cache miss: npm ci (${LOCK_HASH:0:12})"
  npm ci --prefer-offline --no-audit --fund=false
  rm -rf "${NM_CACHE_DIR}"
  mkdir -p "${NM_CACHE_DIR}"
  cp -a node_modules "${NM_CACHE_DIR}/node_modules"
  touch "${LOCK_STAMP}"
fi

# Playwright OS libs — once per Jenkins image (root)
DEPS_STAMP="${CACHE_ROOT}/.playwright-os-deps-chromium"
if [ ! -f "${DEPS_STAMP}" ]; then
  if [ "$(id -u)" = "0" ] && command -v apt-get >/dev/null 2>&1; then
    npx playwright install-deps chromium && touch "${DEPS_STAMP}"
  elif command -v sudo >/dev/null 2>&1 && sudo -n true 2>/dev/null; then
    sudo -n env "PATH=${PATH}" npx playwright install-deps chromium && touch "${DEPS_STAMP}"
  else
    echo "NOTE: run once as root: npx playwright install-deps chromium && touch ${DEPS_STAMP}"
  fi
else
  echo "Cache hit: playwright OS deps"
fi

# Chromium ~170MB — once per @playwright/test version (PLAYWRIGHT_BROWSERS_PATH)
PW_PKG_VER=$(node -p "require('@playwright/test/package.json').version")
PW_STAMP="${PW_CACHE}/.installed-chromium-pw-${PW_PKG_VER}"
HAS_CHROMIUM=$(find "${PW_CACHE}" -maxdepth 1 -type d -name 'chromium-*' 2>/dev/null | head -1 || true)
if [ -f "${PW_STAMP}" ] && [ -n "${HAS_CHROMIUM}" ]; then
  echo "Cache hit: Chromium (Playwright ${PW_PKG_VER}), skip download"
else
  echo "Cache miss: downloading Chromium (Playwright ${PW_PKG_VER})..."
  npx playwright install chromium
  touch "${PW_STAMP}"
fi
du -sh "${PW_CACHE}" 2>/dev/null || true

npm run test:htmlpage
'''
      }
    }
  }

  post {
    always {
      script {
        def status = currentBuild.currentResult ?: 'SUCCESS'
        def ok = (status == 'SUCCESS')
        def badgeColor = ok ? '#16a34a' : '#dc2626'
        def badgeText = ok ? 'PASSED' : status

        sh '''
          set -e
          rm -f lapus-e2e-reports.tgz || true
          PARTS=""
          for d in playwright-report reports test-results; do
            if [ -d "$d" ]; then
              PARTS="$PARTS $d"
            fi
          done
          if [ -n "$PARTS" ]; then
            tar czf lapus-e2e-reports.tgz $PARTS
          fi
        '''

        def bodyHtml = """
          <html>
            <body style="margin:0;padding:24px;background:#f8fafc;font-family:system-ui,Segoe UI,Roboto,Helvetica,Arial,sans-serif;color:#0f172a;">
              <div style="max-width:720px;margin:0 auto;background:#fff;border:1px solid #e2e8f0;border-radius:12px;overflow:hidden;box-shadow:0 10px 30px rgba(15,23,42,.06);">
                <div style="padding:20px 24px;border-bottom:1px solid #e2e8f0;background:linear-gradient(135deg,#eff6ff,#fff);">
                  <div style="display:inline-block;padding:6px 10px;border-radius:999px;background:${badgeColor};color:#fff;font-size:12px;font-weight:700;">${badgeText}</div>
                  <h1 style="margin:12px 0 6px;font-size:20px;line-height:1.25;">Lapus Playwright report (htmlpage)</h1>
                  <p style="margin:0;color:#475569;font-size:14px;">Job: <b>${env.JOB_NAME}</b> · Build: <b>#${env.BUILD_NUMBER}</b></p>
                </div>
                <div style="padding:20px 24px;font-size:14px;line-height:1.6;color:#334155;">
                  <p style="margin:0 0 12px;"><b>Attachment</b></p>
                  <ul style="margin:0 0 16px;padding-left:18px;">
                    <li><code>playwright-report/index.html</code> after extracting <code>lapus-e2e-reports.tgz</code>.</li>
                    <li><code>reports/</code> — site summaries.</li>
                    <li><code>test-results/</code> — traces / screenshots on failure.</li>
                  </ul>
                  <p style="margin:0;color:#64748b;font-size:13px;">Console: <a href="${env.BUILD_URL}console" style="color:#2563eb;">${env.BUILD_URL}console</a></p>
                </div>
              </div>
            </body>
          </html>
        """

        try {
          if (fileExists('lapus-e2e-reports.tgz')) {
            emailext(
              subject: "[${env.JOB_NAME} #${env.BUILD_NUMBER}] Playwright htmlpage — ${badgeText}",
              to: 'queshouhui12@126.com',
              mimeType: 'text/html',
              attachmentsPattern: 'lapus-e2e-reports.tgz',
              body: bodyHtml
            )
          } else {
            emailext(
              subject: "[${env.JOB_NAME} #${env.BUILD_NUMBER}] Playwright htmlpage — ${badgeText} (no report archive)",
              to: 'queshouhui12@126.com',
              mimeType: 'text/html',
              body: bodyHtml + '<p style="margin:16px 0 0;color:#b45309;font-size:13px;">No <code>lapus-e2e-reports.tgz</code> was produced.</p>'
            )
          }
        } catch (err) {
          echo "WARNING: emailext failed — configure SMTP under Manage Jenkins → System: ${err}"
        }
      }
    }
  }
}
