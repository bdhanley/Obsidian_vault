---
recording_id: n6jjEb5d
duration: 4m22s
created_at: 2026-08-19
updated_at: 2026-08-19

---
# Debugging Jenkins login failure direct vs proxy with OIDC/Entra SSO

Date: 2026-08-19

## Transcript

From about the second I finished my design document, until the end of the week, I have to focus on my project: figuring out why a user can access our Jenkins instance through a proxy connection, but a direct connection to it, they cannot log in. Both URLs have separate subdomains, but the main domain is the same, but the subdomains are different. And the Jenkins instance is pointed at an Active Directory domain controller for the proxy that's using the proxied subdomain. I don't know if that makes a difference. And Microsoft Entra is also used for single sign-on as well. So tomorrow I should read through the thread that the user posted, because it looks like they're trying to do something strange with the Jenkins API or trying to log in with a token. I believe OIDC would expire anyways, and the Jenkins instance is set to log them out after an hour, which is maybe why they're using the API to log in. It restarts every day, which would make sense if it's just happening because whatever they're using is not also supplying OIDC. So, first of all, I need to look up tomorrow. I need to research how OIDC works at a very high level. Like, is it giving the user a browser cookie that allows them to sign in to multiple things? Like, how is it actually allowing the user single sign-on for all these different web services? From there, I have to look at how traffic is configured. So I have to find the repo that contains the traffic configuration, because according to that Jenkins host, accessing it through the proxy domain adds an error saying that the proxy is misconfigured. So I have to look into the proxy tomorrow. I should try to automate that myself: connect with the API, try and do what this user is doing after going through the Slack thread where the user detailed their issue. And if I can't reproduce or figure out or find evidence of something that's wrong, then I'll have to ask further questions about what the user is trying to do. After I—if I do find evidence that something is incorrect, most likely it has nothing to do with our team since this involves single sign-on. So then I'll have to reach out to the teams responsible for maintaining that to get them to help us solve our issue.

## Todos

- [ ] Review the user’s Slack/thread details end-to-end and summarize their exact login/API/token flow, timelines, and error messages (proxy URL vs direct URL). #Voicenote
- [ ] Research OIDC at a high level (tokens, cookies/sessions, expiry/refresh, SSO flow with Microsoft Entra) and note what would break when using API/token-only access. #Voicenote
- [ ] Verify Jenkins auth configuration: Active Directory integration, Microsoft Entra/OIDC settings, session timeout (1 hour), and daily restart behavior; document how these affect proxied vs direct subdomain logins. #Voicenote
- [ ] Identify and open the repository/configuration that defines traffic/proxy routing for the Jenkins subdomains; trace differences between proxied and direct URL paths. #Voicenote
- [ ] Investigate the proxy-side issue indicated by the “proxy is misconfigured” error: collect logs/config, confirm headers (Host/X-Forwarded-*), TLS, and callback/redirect URIs for OIDC. #Voicenote
- [ ] Reproduce the issue: attempt login via direct URL and via proxy URL; record outcomes, redirects, and auth errors. #Voicenote
- [ ] Automate a minimal reproduction of the user’s API approach (token/API calls) to validate whether missing OIDC/session context causes logout or failure, especially after restart. #Voicenote
- [ ] If reproduction fails or user intent is unclear, compile specific follow-up questions (what API endpoints, which credential type, expected behavior, frequency, and whether they use browser vs CI client). #Voicenote
- [ ] If evidence points to SSO/identity configuration rather than your team’s scope, escalate with a concise report to the teams owning Entra/OIDC and AD, including repro steps, logs, and suspected misconfigurations. #Voicenote

# Related notes

- [[2026-08-19 Task list post chat, work on proposal, do shift, check on guy]]
- [[2026-08-17 Godot skeleton repos for first- and third-person controller boilerplate]]
- [[2026-08-17 Breath of the Wild-style windy grass shader using vertex colors and noise]]