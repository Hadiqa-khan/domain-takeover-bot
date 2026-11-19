# Domain Takeover Bot

A GitHub Actions workflow that finds subdomains for domains you own and checks them for subdomain takeover.

I built this to keep an eye on forgotten DNS records on domains I own, so a dangling record gets caught before someone else finds it. You can run it on demand against your own domains.

## The problem it catches

A subdomain takeover happens when a DNS record still points at a third-party service (a deleted S3 bucket, a removed Heroku app, an unclaimed GitHub Pages site) that you no longer control. Anyone who claims that resource on the provider's side can serve their own content on your subdomain: phishing pages, cookie theft scoped to your parent domain, or content that looks like it comes from you.

These records pile up quietly as teams spin services up and down, which is why it's worth checking on a schedule rather than once.

## How it works

1. **Validate input.** The root domains you enter are checked against a strict domain-name pattern before anything runs.
2. **Discover.** [subfinder](https://github.com/projectdiscovery/subfinder) collects subdomains from passive sources.
3. **Check.** [Subdominator](https://github.com/Stratus-Security/Subdominator) tests each subdomain for fingerprints of services that can be taken over.
4. **Report.**
   - Every run writes a summary (subdomains found, possible takeovers) to the Actions run page.
   - The subdomain inventory and any findings are uploaded as a run artifact for 7 days.
   - If something is flagged, the workflow opens a GitHub issue, or comments on the existing one instead of opening duplicates.
   - Slack alerts are optional.

## Keep your results private

On a public repository, anyone can open a workflow run and see the domains you entered, the run summary and any issues it opens. I found this out on my first real scan. So:

- **Run it from a private copy.** Use **Use this template** (or GitHub's import tool) to create a *private* repository with this workflow. A fork of a public repo can't be made private.
- **If it does run on a public repo,** the workflow notices and holds back the details. The run summary shows counts only, and no issue or results file is created. Findings still go to Slack if you turn that on.
- Either way, the domains you type are shown on the run page, so treat public runs as public.

## How to run it

1. Create a private copy of this repo (see above). The workflow lives in `.github/workflows/domain.yml`.
2. Go to **Actions → Subdomain Takeover Scan → Run workflow**.
3. Enter your root domains, comma-separated, for example `example.com, example.org`.
4. Optional: to get Slack alerts, add an incoming webhook URL as a repository secret named `SLACK_WEBHOOK_URL` and tick **Send Slack alerts** when you run it.

**Only scan domains you own or are authorized to test.**

## What the output looks like

The run summary (illustrative, using example domains):

```
Subdomain takeover scan
- Subdomains discovered: 42
- Possible takeovers: 1

Verify each finding manually before acting:

    old-docs.example.com  ->  unclaimed GitHub Pages site
```

Treat every finding as a lead, not a verdict. Confirm the DNS record and the provider-side resource before you report or fix anything.

## Security choices in the workflow

The workflow handles untrusted input (the domains you type, and DNS data from the internet), so I treated it like any other code that runs with credentials:

- **No script injection.** The domain input reaches the shell as an environment variable, never interpolated into the script, and is validated before use.
- **Least-privilege token.** Permissions default to none; the job only gets `contents: read` and `issues: write`.
- **Pinned and verified tools.** subfinder and Subdominator are pinned to exact releases and checked against SHA-256 hashes before they run. The one third-party action (`actions/upload-artifact`) is pinned to a full commit SHA.
- **Safe message building.** Findings are written to files and Slack payloads are built with `jq`, so scan data can't break or alter the issue or the alert.
- **No results in public places.** Tool output goes to files, not the job log, and on public repos the summary, issue and results file hold back the details.
- **Fewer dependencies.** Issues are created with the built-in `gh` CLI and Slack messages are sent with `curl`, instead of extra third-party actions.

## Ideas for next steps

- Add [nuclei](https://github.com/projectdiscovery/nuclei)'s takeover templates as a second checker. Subdominator's last release was in 2024, so its fingerprints may miss newer services.
- Compare each run with the previous inventory and alert on new subdomains, not just takeovers.
- Add a weekly `schedule` trigger with the domain list stored as a repository variable, for anyone running it continuously.
