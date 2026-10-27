# Security Policy

This repository contains documentation only; there is no software to patch. The main security risk is **accidental publication** of real infrastructure details.

## What to report

- Real secrets: passwords, API keys, tokens, private keys, recovery keys, VPN configs.
- Real public IPs, hostnames, domains, usernames, serial numbers, or MAC addresses.
- Screenshots or command output that reveal a real environment.
- Guidance that is unsafe in a way that could cause lockout or data loss if followed as written.

## How to report

Use GitHub private vulnerability reporting: open the repository's **Security** tab and choose **Report a vulnerability**.

Please **do not open a public issue, discussion, or pull request** that quotes the sensitive data. Doing so republishes it.

Include:

- the file and line, or commit, where the data appears;
- what kind of data it is (without repeating the value if you can avoid it).

## What happens next

- The maintainer will acknowledge the report as soon as practical.
- Exposed secrets are treated as compromised and rotated, regardless of whether history is rewritten.
- If needed, the content is removed from the working tree and the maintainer decides whether a history rewrite is required.
- Reporters are credited if they wish.
