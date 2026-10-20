#!/usr/bin/env node
'use strict';

const http = require('http');
const fs = require('fs');
const path = require('path');
const { exec } = require('child_process');

const TIMEOUT_MS = 5 * 60 * 1000; // 5 minutes
const BASE_PORT = 2345;
const MAX_PORT_ATTEMPTS = 5;

// ── Parse stdin ──────────────────────────────────────────────────────────────

function readStdin() {
  return new Promise((resolve) => {
    let data = '';
    process.stdin.setEncoding('utf8');
    process.stdin.on('data', (chunk) => { data += chunk; });
    process.stdin.on('end', () => resolve(data));
  });
}

function extractPlan(raw) {
  const match = raw.match(/<comprendo_data>([\s\S]*?)<\/comprendo_data>/);
  if (!match) return null;
  try {
    return JSON.parse(match[1].trim());
  } catch (e) {
    console.error('Failed to parse JSON from <comprendo_data>:', e.message);
    return null;
  }
}

// ── HTTP server (zero dependencies) ──────────────────────────────────────────

function buildServer(plan) {
  const uiPath = path.join(__dirname, 'ui.html');
  let uiTemplate;
  try {
    uiTemplate = fs.readFileSync(uiPath, 'utf8');
  } catch (e) {
    console.error('Could not read ui.html:', e.message);
    process.exit(2);
  }

  const injectedHtml = uiTemplate.replace(
    '__COMPRENDO_DATA__',
    JSON.stringify(plan).replace(/</g, '\\u003c').replace(/>/g, '\\u003e')
  );

  let resolveDecision;
  const decisionPromise = new Promise((resolve) => { resolveDecision = resolve; });

  const server = http.createServer((req, res) => {
    if (req.method === 'GET' && (req.url === '/' || req.url === '')) {
      res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
      res.end(injectedHtml);
      return;
    }

    if (req.method === 'POST' && req.url === '/decision') {
      let body = '';
      req.on('data', (chunk) => { body += chunk; });
      req.on('end', () => {
        let approved = false;
        try {
          const parsed = JSON.parse(body);
          approved = parsed.approved === true;
        } catch {}
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({ ok: true }));
        resolveDecision(approved);
      });
      return;
    }

    res.writeHead(404);
    res.end('Not found');
  });

  return { server, decisionPromise };
}

// ── Port binding with fallback ───────────────────────────────────────────────

function tryListen(server, port) {
  return new Promise((resolve, reject) => {
    server.once('error', reject);
    server.listen(port, () => {
      server.removeListener('error', reject);
      resolve(port);
    });
  });
}

async function startServer(server) {
  for (let i = 0; i < MAX_PORT_ATTEMPTS; i++) {
    const port = BASE_PORT + i;
    try {
      await tryListen(server, port);
      return port;
    } catch (err) {
      if (err.code !== 'EADDRINUSE' || i === MAX_PORT_ATTEMPTS - 1) {
        console.error(`Could not bind to ports ${BASE_PORT}-${BASE_PORT + MAX_PORT_ATTEMPTS - 1}.`);
        process.exit(2);
      }
    }
  }
}

// ── Open browser ─────────────────────────────────────────────────────────────

function openBrowser(url) {
  const cmd = process.platform === 'darwin' ? 'open' : 'xdg-open';
  exec(`${cmd} "${url}"`, (err) => {
    if (err) {
      console.error(`Could not open browser automatically. Please visit: ${url}`);
    }
  });
}

// ── Main ─────────────────────────────────────────────────────────────────────

async function main() {
  const args = process.argv.slice(2);

  if (args[0] !== 'review') {
    console.error('Usage: comprendo review');
    console.error('  Pipe a <comprendo_data> JSON block to stdin.');
    process.exit(2);
  }

  if (process.stdin.isTTY) {
    console.error('No plan data on stdin. Pipe the <comprendo_data> block to this command.');
    process.exit(2);
  }

  const raw = await readStdin();
  const plan = extractPlan(raw);
  if (!plan) {
    console.error('No valid <comprendo_data> block found in input.');
    process.exit(2);
  }

  const { server, decisionPromise } = buildServer(plan);
  const port = await startServer(server);
  const url = `http://localhost:${port}`;

  console.error(`\n  Comprendo review UI running at ${url}\n`);
  openBrowser(url);

  const timeout = setTimeout(() => {
    console.error('\n  Timed out waiting for decision. Rejecting to be safe.\n');
    server.close();
    process.exit(1);
  }, TIMEOUT_MS);

  const approved = await decisionPromise;

  clearTimeout(timeout);
  server.close();

  if (approved) {
    console.error('  Approved. Claude will now implement.\n');
    process.exit(0);
  } else {
    console.error('  Rejected. Claude will stop.\n');
    process.exit(1);
  }
}

main().catch((err) => {
  console.error('Comprendo error:', err);
  process.exit(2);
});
