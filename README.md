# Battle Pets — Fight a Friend server

This is a real HTTP multiplayer server, separate from the Netlify static website. It uses short polling so no paid WebSocket infrastructure or npm dependencies are needed.

## Deploy
1. Create a GitHub repository containing these three server files (server.js, package.json, render.yaml).
2. On Render.com choose **New > Blueprint**, connect the repository, then deploy the blueprint. On free Render hosting the server may sleep when inactive; the first connection may take longer.
3. Copy the Render public server URL (such as `https://your-app.onrender.com`).
4. Redeploy the included updated website ZIP to your existing Netlify site.
5. Open your Battle Pets game, choose Trainer Arena > Fight a Friend, select **Server settings** and paste the Render URL on **both** players' devices. The game remembers it in the browser.
6. Each player uses the same four-digit code and clicks Join Match. The match begins after both join.

The server keeps rooms in memory; restarts reset active rooms. Codes are not private authentication: share them only with the person you intend to battle. Don't share personal information in a trainer name. Only one match can occupy each code at a time. The mode is a lightweight **one-pet-versus-one-pet** multiplayer battle, not a full six-pet party battle. Combat uses quick and strong attacks, with the existing five-element type matchup rules. This is separate from story battles and does not award XP or coins. Pet stats are client-supplied, so it isn't cheat-proof. Do not expose the server's URL to random people if you want a private server. For a public production service, add rate limiting, authentication, a persistent datastore and authoritative pet verification.

## Local test
Run `npm start` and visit `http://localhost:3000/health`.

Set `ALLOWED_ORIGINS` to the Netlify address or a comma-separated list of allowed site origins. Set `PORT` if your host requires it.
