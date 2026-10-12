---
recording_id: kJP2pVCo
duration: 3m33s
created_at: 2026-08-19
updated_at: 2026-08-19

---
# Jenkins Active Directory plugin LDAP edge-case causing API token errors

Date: 2026-08-19

## Main points

- Tested multiple browsers and used developer tools to observe behavior when issuing Jenkins session cookies across two separate domains.
- No major issues found; the older Jenkins instance reported a more severe reverse-proxy misconfiguration warning.
- Checked LDAP server records because the Jenkins Active Directory plugin relies on LDAP lookups for users/groups (not OIDC or SAML).
- Attempted to reproduce an API token error by changing various settings; concluded it is likely an edge case.
- Suspected causes include an issue specific to one Active Directory server and/or an LDAP security vulnerability in version 2.41.1 that may allow enumeration of groups/hosts and potential user login via LDAP.
- Reviewed Splunk logs: Jenkins repeatedly enumerated valid groups; intermittent failures occurred.
- Observed failure pattern: after an LDAP failure, Jenkins immediately retried authentication too quickly, triggering an LDAP error indicating a bind is required first ("You have to bind first").
- Overall hypothesis: a strange edge case in the Jenkins Active Directory plugin version and/or a rare edge case involving token groups while collecting SIDs/SSIDs.
- Possible timing/permission propagation issue: permissions may not appear until entered on another system, after which the browser session cookie is established.

## Transcript

So I've just been out there using many different browsers, going through dev tools to look at what was happening each time I'd issue the session cookie with Jenkins on both, um, both separate domains. I didn't find anything too bad except that Jenkins complained on the older one about a worse proxy misconfiguration. Besides that, I looked up the server records for LDAP because it turns out that the Active Directory plugin for Jenkins doesn't do anything interesting like OIDC or SAML, so it just literally looks up users and groups in LDAP. So at that point, I even tried to reproduce the API token error, um, I tried to mess with different things to make it work, but ultimately I think this is just an edge case: either on the Active Directory server that it kept happening on, or it has to do with the apparent LDAP security vulnerability in 2.41.1 that allows you to enumerate groups and hosts and potentially log in as a user on that LDAP. And I spent a bunch of time going through Splunk logs and found it consistently exhausting groups that were valid, and then sometimes it would fail, and then it would immediately try to authenticate again but it did it too fast, and there would be another error because LDAP would say, "You have to bind first." So altogether, I believe this is some strange edge case either with that version of the Active Directory Jenkins plugin or another possibility is a super weird edge case with token groups, since it has to basically walk through that entire thing and collect SSIDs. So it's possible sometimes it just, um, does it a bit differently and then the appropriate permissions don't come through until they're entered on the other one, and then the session cookie is then in the browser. So, yeah.

# Related notes

- [[2026-08-19 Debugging Jenkins login failure direct vs proxy with OIDCEntra SSO]]