# FrontDesk AI — how to put it online (GitHub + Netlify, no coding)

You need: a free GitHub account, your Netlify account, Chrome or Edge. About 15 minutes.

## Part 1 — Put the files on GitHub
1. Unzip FrontDesk-AI.zip to E:\ (right-click → Extract All → choose E:\). You get the folder E:\FrontDesk-AI
2. Go to https://github.com and sign in (or Sign up — free).
3. Top-right "+" → "New repository".
   - Repository name: frontdesk-ai
   - Choose "Private" (or Public — both work)
   - Do NOT tick "Add a README"
   - Click "Create repository".
4. On the next page click the link "uploading an existing file".
5. Open E:\FrontDesk-AI in File Explorer. Press Ctrl+A (select everything INSIDE the folder),
   then drag it all onto the GitHub page. Wait until all files are listed (68 files).
   Important: drag the CONTENTS of the folder, not the folder itself — package.json must be at the top level.
6. Scroll down, click the green "Commit changes". Wait until the file list appears.

## Part 2 — Connect Netlify (it builds the site for you)
1. Go to https://app.netlify.com → "Add new site" → "Import an existing project".
2. Choose "GitHub", allow Netlify to access GitHub, pick the repository frontdesk-ai.
3. Netlify reads the settings from netlify.toml. You should see:
   Build command: npm run build · Publish directory: dist · Functions: netlify/functions
   Do not change anything. Click "Deploy".
4. Wait 1–3 minutes until it says "Published". Your site address looks like https://something.netlify.app
   (Site configuration → Change site name to pick e.g. frontdesk-brightroots).

## Part 3 — (optional) add the DeepSeek key — NEVER paste it in a chat
1. Netlify → your site → Site configuration → Environment variables → Add a variable.
2. Key: AI_API_KEY   Value: your DeepSeek key   (mark it secret if offered) → Create.
3. Optional second variable: AI_MODEL = deepseek-chat
4. Deploys → "Trigger deploy" → "Deploy site" (the key is only picked up by a new deploy).
Without the key everything still works — answers come from the clinic file.

## Part 4 — check it works (2 minutes)
Open these on your phone and computer (replace YOUR-SITE):
- https://YOUR-SITE.netlify.app/?q=How%20much%20is%20a%20root%20canal%3F   → shows Rs 6,500 to Rs 9,000
- https://YOUR-SITE.netlify.app/staff   → the ticket appears; the yellow bar says "Shared server storage (Netlify Blobs)"
  (if it says "Local demo mode" the server part did not deploy — send me a screenshot of Deploys → the latest deploy log)
- Reply from /staff → the reply appears in the patient chat within 3 seconds.
- With the key: the root-canal answer shows "Source: …" and no "Answered from the clinic file" label.
- https://YOUR-SITE.netlify.app/?nokey=1 → always shows "Answered from the clinic file".

## Updating later
Change files on GitHub (open the file → pencil icon → edit → Commit), or upload new files the same way.
Netlify rebuilds automatically after every commit.
