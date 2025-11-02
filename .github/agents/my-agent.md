---
name: bob
description: bob is bags assistance, designing the digital infastructure for neobank finance.
name: Help Bot
on:
  issues:
    types: [opened, edited]
  pull_request:
    types: [opened, edited]
  issue_comment:
    types: [created]

permissions:
  contents: read
  issues: write
  pull-requests: write

jobs:
  help-bot:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout (for KB file)
        uses: actions/checkout@v4

      - name: Help Bot Logic
        uses: actions/github-script@v7
        with:
          script: |
            const core = require('@actions/core');
            const github = require('@actions/github');
            const ctx = github.context;

            // ---- Load KB (optional) ----
            // Create .github/help-bot.yml with simple Q/A pairs (see below).
            const fs = require('fs');
            let kb = [];
            try {
              const yaml = require('js-yaml');
              const raw = fs.readFileSync('.github/help-bot.yml', 'utf8');
              kb = yaml.load(raw) || [];
            } catch (e) {
              // no kb is fine
            }

            // ---- Helpers ----
            const isIssue = !!ctx.payload.issue;
            const isPR = !!ctx.payload.pull_request;
            const number = isIssue ? ctx.payload.issue.number : ctx.payload.pull_request?.number;
            const body = (isIssue ? ctx.payload.issue.body : ctx.payload.pull_request?.body) || '';
            const commentBody = ctx.payload.comment?.body || '';
            const author = (isIssue ? ctx.payload.issue.user.login : ctx.payload.pull_request?.user.login) || ctx.payload.comment?.user?.login;

            async function comment(msg) {
              if (!number) return;
              const repo = ctx.repo;
              if (isIssue || ctx.eventName === 'issue_comment') {
                return github.rest.issues.createComment({ ...repo, issue_number: number, body: msg });
              } else if (isPR) {
                return github.rest.issues.createComment({ ...repo, issue_number: number, body: msg });
              }
            }

            async function addLabels(labels) {
              if (!number || !labels?.length) return;
              const repo = ctx.repo;
              await github.rest.issues.addLabels({ ...repo, issue_number: number, labels });
            }

            // ---- Auto-greet on open ----
            if (ctx.eventName !== 'issue_comment' && (isIssue || isPR)) {
              const greet = [
                `Hi @${author}! I'm the repo helper 🤖.`,
                `Type \`/help\` for available commands.`,
                `Search the knowledge base with \`/kb <keywords>\`.`,
                `Maintainers can use \`/assign me\`, \`/close\`, \`/label bug|feature|question\`.`
              ].join('\n');
              await comment(greet);

              // naive auto-labeling
              const lower = (body || '').toLowerCase();
              const labels = [];
              if (/\bbug|error|crash|fail|exception\b/.test(lower)) labels.push('bug');
              if (/\bfeature|enhancement|improve|idea\b/.test(lower)) labels.push('feature');
              if (/\bquestion|how\b/.test(lower)) labels.push('question');
              if (labels.length) await addLabels(labels);
              return;
            }

            // ---- Commands via comments ----
            if (ctx.eventName === 'issue_comment') {
              const cmd = commentBody.trim();

              if (/^\/help\b/i.test(cmd)) {
                await comment(
                  [
                    `**Help Bot Commands**`,
                    `• \`/kb <keywords>\` — search KB entries.`,
                    `• \`/assign me\` — assign the commenter (maintainers only).`,
                    `• \`/label <name>[,name]\` — add labels (maintainers only).`,
                    `• \`/close\` — close the thread (maintainers only).`,
                    ``,
                    `KB file: \`.github/help-bot.yml\``
                  ].join('\n')
                );
                return;
              }

              // /kb search
              const kbMatch = cmd.match(/^\/kb\s+(.+)/i);
              if (kbMatch) {
                const q = kbMatch[1].toLowerCase();
                const results = kb
                  .map((entry, i) => ({ i, score: (entry.q || '').toLowerCase().includes(q) || (entry.a || '').toLowerCase().includes(q) }))
                  .filter(r => r.score)
                  .map(r => kb[r.i])
                  .slice(0, 3);

                if (!results.length) {
                  await comment(`No KB hits for **${q}**. Maintainers can add entries in \`.github/help-bot.yml\`.`);
                } else {
                  const msg = results.map(e => `**Q:** ${e.q}\n**A:** ${e.a}`).join('\n\n---\n\n');
                  await comment(msg);
                }
                return;
              }

              // Repo membership check for privileged commands
              async function isCollaborator(login) {
                try {
                  await github.rest.repos.checkCollaborator({ ...ctx.repo, username: login });
                  return true;
                } catch {
                  return false;
                }
              }

              // /assign me
              if (/^\/assign\s+me\b/i.test(cmd)) {
                if (!(await isCollaborator(author))) return comment(`Sorry @${author}, only collaborators can use \`/assign me\`.`);
                await github.rest.issues.addAssignees({ ...ctx.repo, issue_number: number, assignees: [author] });
                await comment(`Assigned to @${author}.`);
                return;
              }

              // /label one,two
              const labelMatch = cmd.match(/^\/label\s+(.+)/i);
              if (labelMatch) {
                if (!(await isCollaborator(author))) return comment(`Only collaborators can label.`);
                const labels = labelMatch[1].split(/[,\s]+/).filter(Boolean);
                await addLabels(labels);
                await comment(`Added label(s): ${labels.join(', ')}`);
                return;
              }

              // /close
              if (/^\/close\b/i.test(cmd)) {
                if (!(await isCollaborator(author))) return comment(`Only collaborators can close.`);
                await github.rest.issues.update({ ...ctx.repo, issue_number: number, state: 'closed' });
                await comment(`Closed by @${author}.`);
                return;
              }
            }


# My Agent

Describe what your agent does here... he helps and organizes experiences. navigates the user to the rigth destination and is very helpful at orchastrating direction on where to buy memberships and mentorships / portfolio tracking
