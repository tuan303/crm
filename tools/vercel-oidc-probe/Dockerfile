# syntax=docker/dockerfile:1
FROM node:26.13.0-alpine3.23
WORKDIR /app
COPY <<'CRM_PROBE_SERVER' /app/server.mjs
import { readFileSync } from 'node:fs';
import { createServer } from 'node:http';

function startupTokenFileReadableNonempty() {
  const tokenFile = process.env.VERCEL_OIDC_TOKEN_FILE;
  if (!tokenFile) return false;
  try {
    return readFileSync(tokenFile, 'utf8').trim().length > 0;
  } catch {
    return false;
  }
}

// Capture these before listening; later requests cannot alter the startup result.
const startup = Object.freeze({
  startupEnvTokenPresent: Boolean(process.env.VERCEL_OIDC_TOKEN?.trim()),
  startupTokenFileConfigured: Boolean(process.env.VERCEL_OIDC_TOKEN_FILE),
  startupTokenFileReadableNonempty: startupTokenFileReadableNonempty(),
});

const port = Number(process.env.PORT ?? 3000);
if (!Number.isInteger(port) || port < 1 || port > 65535) process.exit(1);

const server = createServer((request, response) => {
  request.resume();
  const tokenHeader = request.headers['x-vercel-oidc-token'];
  const body = JSON.stringify({
    ...startup,
    requestOidcHeaderPresent:
      typeof tokenHeader === 'string' && tokenHeader.trim().length > 0,
  });
  response.writeHead(200, {
    'Content-Type': 'application/json; charset=utf-8',
    'Content-Length': Buffer.byteLength(body),
    'Cache-Control': 'no-store',
    'X-Content-Type-Options': 'nosniff',
  });
  response.end(body);
});

// Keep errors silent: never print credentials, environment values, or file paths.
server.on('error', () => process.exit(1));
server.listen(port, '0.0.0.0');
for (const signal of ['SIGTERM', 'SIGINT']) {
  process.once(signal, () => {
    server.close(() => process.exit(0));
    setTimeout(() => process.exit(0), 3000).unref();
  });
}
CRM_PROBE_SERVER
ENV PORT=3000
USER 1000:1000
EXPOSE 3000
CMD ["node", "server.mjs"]
