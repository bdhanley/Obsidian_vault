---
recording_id: vv7F0M5w
duration: 22m36s
created_at: 2026-08-21
updated_at: 2026-08-21
#Meeting
---
# Jenkins migration to new VM with K3s volumes, TLS, and ingress updates

Date: 2026-08-21

## Meeting Report

## Overview
The team migrated Jenkins from an older VM to a new VM to upgrade the Oracle version and remediate risk after a secrets exposure incident, preserving key Jenkins state via K3s volumes. Job history was transferred and verified via “reload configuration from disk,” and the final cutover required merging K3s ingress/certificate changes and updating DNS/host routing.

## Migration rationale and scope
- Move to a new VM to:
  - Upgrade to a later Oracle version.
  - Address a prior security incident where an Ansible pipeline exposed secrets; environments were treated as tainted, making a clean migration necessary rather than only rotating secrets.
- Goal: preserve Jenkins state between runs while moving jobs/history and restoring equivalent functionality on the new host.

## Preserved state (K3s volumes) and what was added
- Preserved items discussed:
  - Notes, secrets, config history
  - Email templates
  - Kit mirror data used by the controller
  - Logs
  - Garbage collection (GC) logs (on a separate volume)
  - Monitoring state so restarts retain prior monitoring context
- Session script scope: touches only logs and jobs.
- Deployment is extensible: additional volumes can be added over time as needs evolve; volumes must be described/mounted appropriately for the controller/container.

## Deployment mechanics, verification, and debugging
- Job history transfer process:
  - Transfer job history to destination -> unzip -> deploy into the target volumes.
- Verification steps mentioned:
  - Use Jenkins “reload configuration from disk” to discard in-memory data and reload from filesystem (used as a first-time test since no builds were running).
  - Confirm job history appears and matches production.
  - After redeploys, confirm pods have restarted recently (uptime resets) and are not erroring.
- Operational notes:
  - There is no strong monitoring administrative plane; much of the operational workflow is via `kubectl`.
  - README includes commands for debugging/verification (e.g., retrieving logs, stopping pods—K3s will recreate them based on host configuration).
  - Logs are also available in Splunk, but local extraction may still be needed for machine-level debugging.

## Repo/layout structure and reuse approach
- Configuration structure:
  - Folder per team (e.g., lotus, northstar, and the team’s own folder).
  - Folder per deployment (per VM), leading to some duplication due to unique per-instance configuration.
  - Common patches/resources centralized where possible (certificates, ESS secret definitions, shared resources).
- Splunk in K3s:
  - Splunk is configured through K3s configuration files; two Splunk pods exist (agent + service).
  - Templates are referenced from a shared “Splunk Auto Templates” location.

## Traffic, TLS certificates, and where configuration lives
- Traffic configuration location:
  - Traffic config is in a separate repo/app from the K3s app (two-app setup).
  - Some templates (e.g., Nginx-related, secrets retrieval) are in the K3s app, while some configuration may live in Ansible.
- Ansible’s role:
  - Installs and initially configures K3s.
  - Configures how TLS certs are pulled; the actual cert material is stored as K3s secrets.
- TLS secrets clarification (traffic namespace):
  - Multiple secrets exist, including:
    - A “theory” certificate.
    - A default TLS secret for the AWS host (e.g., `redwoodaws.ad.ia.com`).
  - `kubectl get secret -n traffic` was used to locate the relevant secrets.

## Cutover: domain/DNS routing, ingress, and merge request
- DNS/hostname strategy:
  - Using a controllable subdomain (e.g., `theory.ia.com`) enables the team to update A records/CNAMEs without IT involvement; propagation observed in minutes.
- Final cutover steps:
  - Merge request in the K3s repo to:
    - Ensure the DRE certificate is installed/used for the Jenkins endpoint.
    - Update ingress to route host traffic (e.g., `gcassist-prod...`) to the new Jenkins instance.
    - Update Jenkins to use the correct external URL to avoid “magic variable” URL mismatches that would otherwise point users/log links to the VM hostname.
  - Note: Certificate secret was already present, but ingress was not previously set up; ingress is per-instance and must explicitly include the hostnames to accept.
  - Builds were disabled during redeploy; request was made to keep the disabled window short and do teaching after merging.

## Action items
- Merge the K3s repo merge request to apply ingress/URL changes for the new Jenkins endpoint (required for redirect/cutover).
- Redeploy after merge and verify pods come up clean (no errors) and Jenkins loads without issues.
- Confirm external access behavior on the new host (expected “not secure”/cert behavior until fully configured) and validate Jenkins URLs point to the intended hostname rather than the VM.
- Re-enable builds after cutover validation to minimize downtime.
- Complete the deferred “teaching” walkthrough after the merge (requested to avoid extending the build-disabled window).

## Main points

- Jenkins deployment uses persistent volumes to preserve state between runs (notes, secrets, config history, email template, kit mirror used by controller, logs).
- Newest deployment adds a separate volume for garbage collection (GC) logs.
- Session script touches only logs and jobs.
- Monitoring persistence was added so container restarts keep prior monitoring state.
- Volumes are editable; additional volumes can be added via a volume patch as deployment evolves.
- Two added items highlighted: GC logs volume and monitoring volume.
- Mounts must be described for the controller/Jenkins container, including the destination paths (default disk location).
- Job history is being transferred to the destination, then unzipped and deployed to the volumes.
- No strong monitoring admin plane exists; operations are largely done via command line using kubectl.
- README includes kubectl commands for debugging/verification (e.g., getting logs, extracting logs from the machine).
- Logs are also available in Splunk, but local extraction may be needed for machine-level debugging.
- Recommended way to stop a container/pod is via kubectl; K3S will recreate it by resyncing from host configuration stored in specific folders.
- Repository/folder structure: folder per team (e.g., lotus, northstar, and their team) and folder per deployment/VM, leading to duplication due to unique configurations.
- Common patches/resources are centralized and reused (e.g., certificates, secret definitions in ESS, other shared resources).
- Splunk is configured via configuration files; deployment includes two Splunk pods (agent and service), using templates from “Splunk Auto Templates.”
- Traffic configuration is not in the K3S repo; it is in a separate repo/app started last year during migration from Chef to K3S.
- Some traffic/Nginx-related templates (including how traffic gets secrets) are in that app; some K3S/traffic configuration may also be in Ansible.
- Ansible installs and initially configures K3S, including configuring how TLS certs are pulled; actual certificates are stored as K3S secrets.
- Kubernetes namespaces are used; to view traffic secrets, specify the “traffic” namespace (otherwise only default namespace shows).
- Two TLS secrets referenced: one for the theory domain and a “default TLS” for the AWS domain (redwoodaws.ad.ia.com).
- Reason for migrating Jenkins assets to a new VM: upgrade to a later Oracle version and response to a security leak where an Ansible pipeline exposed secrets; environments considered tainted so secrets change alone was insufficient.
- After deploying on the new server, Jenkins initially shows no job history until “Reload configuration from disk” is used to reload from filesystem (safe since no builds running).
- After reload, job history appears and matches prior production history, confirming successful transfer.
- Domain management approach: use their own domain (theory.ia.com) to self-manage DNS (A records/CNAMEs) without IT intervention; DNS changes propagated quickly (~5 minutes).
- New server access may show “not secure” until the certificate is installed; production works because certificate is installed and Jenkins server URL settings point correctly.
- Need to adjust Jenkins “magic variables”/URL settings to avoid users being redirected to VM-specific addresses or incorrect log links.
- Final cutover requires redirect/ingress changes via a merge request in the K3S repo; requires restart/redeploy to install/configure.
- A request was made to do teaching after merging to minimize time builds remain disabled.
- Merge request includes four changes: install DRE certificate, point server to use gcassist-prod.dre URL, stop using prior URL/file, and update ingress.
- DRE secret object/certificate was already installed previously; missing piece was ingress configuration.
- Ingress is custom per instance; configured to ingress host gcassist-prod for Jenkins and optionally Git mirror (default was AWS host, but can add another).
- Post-deploy validation step: check pods are not erroring; pod uptime indicates recent restart/redeploy (e.g., running for 2 minutes instead of 14 days).
- After redeploy, verify Jenkins login and confirm history reload succeeded without errors.

## Transcript

[00:00:00] Speaker 1: Preserve, um, the preserved state between runs. If you've seen— you've probably seen some of them— but these are the ones with notes, secrets, config history, the email template, the kit mirror so that we don't— which the controller uses, the logs, and I think on the newest deployment we also have— Right, so we also have the garbage collection logs, for which we set up a separate volume. Right, so the session script is touching only notes— sorry, the session script is touching only logs and jobs. Right. Anything else we're not— oh, no, sorry, we also added monitoring so that when you restart the container, the next session is going to keep the monitoring from, like, before, which we weren't doing. Yeah, but this stuff is editable, so as you know, as our deployment is evolving, like, this was the default that we got. You can add more volumes, more stuff that you— that you want to preserve. So we added some extra— basically added a volume patch so we can have— so we can have some more— sorry, no, this is the actual one I'm monitoring for. So we added these two so that we can have the GC logs and the monitoring. Right, so, and then you just have to describe what you're mounting on the controller, on the— on the Jenkins container. The container, sorry, in Jenkins. And then the actual path would be where it's going. It's— so it's on a disk, default location here. So, yeah, so that's one of the things that I— before doing this transfer, I kind of wasn't very worried about it. Okay, so we're transferring over the job history now to the destination. Then it's just going to be unzipped and deployed to those two volumes. Any questions, I guess, or—

[00:03:10] Speaker 2: No, but thank you. This is— this is really good to watch. I think Christine and I have both been kind of looking at K3S a lot lately, so, I mean, she's got more experience working with it than I do, but it's good to kind of see it, like, you doing it, like, a real deployment in person. So thank you.

[00:03:30] Speaker 1: Yeah, so the actual deployment, we are not— we're just going to be— so, okay, so we don't have a good monitoring administrative plane, which is a problem. I think when I was more involved in this— they currently are more in support mode— they were playing with something there, but, like, anything. Pretty much do on the command line with— with— with kubectl. So I have added some commands in here, in the README, which should help you kind of, like, in debugging and verification and stuff, like getting logs, for example. So all the logs are actually also in Splunk, but sometimes, especially if you're debugging something on the machine, you might also need to, like, extract them. If you want to— if you want to stop a container or pod, this is the best way. It will be recreated by K3S. It's just going to resync from whatever the configuration is for the host, which means from here, in our case, from any of these folders. So the other thing is, I think we've talked to you and Christina, so the way we structured with this— we structured this one so we have, like, a folder per— kind of per team. So there's one: lotus, northstar. This is ours. And then there's a folder per deployment. That means per virtual machine. So there is a lot of copying in between because a lot of these configurations are kind of unique. There are some common patches and resources here, which we reuse as much as we can, like, just one for the tea— for the theory. Certificates. This is for defining secrets in ESS, and there's a bunch of others. And we also have an app for the Splunk deployment. So if you've seen on these configuration files, we do configure Splunk, the Splunk pods. There's two Splunk pods, one for the agent and another for the service. But the actual templates are in here. So we refer to this location, Splunk Auto Templates. Yeah, so we do try to reuse stuff as much. We also have some— anything that's common, we put it into this folder, whether it's patches or resources. So, yeah. Does that sound, like, meaningful, Christine and Brian, or—

[00:06:47] Speaker 2: Yeah, absolutely. Is this where the configuration for traffic is located?

[00:06:53] Speaker 1: No. This— the configuration for traffic is located here, in this repo. So there's really two apps. So this is the one which was— which we started off with last year when we swapped over to K3S from Chef. Everything before was driven by Chef. This is one that we set up last year for auto, but this is where traffic is configured. It's in here. It's in— so, uh, let me just— so these would be templates for it, but the actual configuration may actually also— some of it may be in Ansible.

[00:07:35] Speaker 2: Right, okay.

[00:07:36] Speaker 1: I'm sorry. So, um, let's roll back here. And that means, right, because Ansible installs and initially configures K3S, okay, the templates for— some of the common templates are in that. So let's go to— some of the templates are— basically the templates for Nginx read, including how traffic would get secrets and whatnot, is in that app. However, how K3S, including traffic, is configured, may— some of this may be in Ansible. Does this make sense?

[00:08:20] Speaker 2: Yeah, like, does Ansible— is Ansible the one that installs, like, the TLS certificates? Or the TLS, like, private key insert?

[00:08:29] Speaker 1: So, um, Ansible configures how you're going to pull them. The actual cert is in K3S secret, I believe.

[00:08:48] Speaker 2: Right, right, okay.

[00:09:04] Speaker 1: So this is the theory secret, actually. So it's not— it's not here. Yeah, so this is the theory certificate. However, there's another one. That's the self-signed cert for this specific host, right, because if you remember, we basically would have, like, this is for the theory domain, but for the AWS domain there was something else, which— ah, there's the one. I can— I couldn't show— I don't know what I remember because I was looking in this. No, I did this one. Sorry. So this is the actual— this is the other secret for— for the default— for the— what it's called, default TLS, right. So it's not in the— so if I— so if I do this, right, so get secret— because I forgot what the namespace for traffic is. It's just traffic. You have different namespaces in a Kubernetes deployment. And if I don't specify the namespace, it will only show me the default one. There it is. So this is the other one. This is the— so this is the theory one. This is the default one, meaning the one for redwoodaws.ad.ia.com. Does this make sense?

[00:11:02] Speaker 2: Yep, that made absolute sense.

[00:11:04] Speaker 1: Okay, cool. Okay, let me see. Okay, so we are done. Sorry.

[00:11:12] Speaker 3: Yeah, the quick questions. Maybe I'm lack of some context, but just wondering, why are we moving the old assets for Jenkins to the new one? Because I can see from your video studio code, we are having, like, all the duplicated version for both staging assets, prod assets, and the CI staging everything. Just curious.

[00:11:37] Speaker 1: Yeah, so we are moving to a new VM for a couple reasons. We're moving to a later version of Oracle. That's the first. And then the second one, we had a security leak, but not just us, but— and we had a security leak because the Ansible pipeline we were running at the time was exposing secrets, and we had to migrate to a new machine because we didn't really know whether, you know, some bad actor had done something. We couldn't just change the secrets, right. We considered those environments tainted.

[00:12:23] Speaker 3: Mm-hmm, makes sense.

[00:12:24] Speaker 1: Does this make sense? Okay.

[00:12:26] Speaker 3: Yes, yes.

[00:12:28] Speaker 1: Okay, so we deployed. So the first thing I'm going to do, so I'm going to log into— so the new server. So I'm going to— not going to use— so, uh, so this proto2 is the new one. So there— oh, GTS is proto2. Oh, no, sorry, that's because I'm— I'm missing this. Sorry. So this is the actual name. This is the wrong password. Okay. No, so, um, this is the new one, right. As you can see, we haven't run any jobs, so there's probably no history. We deployed successfully, but for Jenkins to pick it up, we'd have to restart the server, or we can do something else. We can go here. We can redo— we can reload configuration from disk. That's going to discard all the loaded data in memory and reload everything from filesystem. So we're not running any builds. We can do this. So this is kind of like a first-time test. So now you see we have history, right. The jobs have history. So this is identical history as what we've had on prod. The history was transferred. Now the last phase is to point, because one of the things I did earlier— one of the things we did earlier, you guys have noticed this. We have this thing— we have our own domain, theory.ia.com, right, and it's point really right now for us, there's only one point that we have for us. There might be another one in the future, but right now we can have our own unique host name, and we can assign that host name by ourselves to different A records or even C names, right. So an A record is an IPv4 address.

[00:14:53] Speaker 2: Oh.

[00:14:55] Speaker 1: Yeah, so this is very useful for us because we can give to the dev team, you know, this host name, and we ourselves can manage it. We don't have to contact IT and get back, whatever, ask them to change that or even to delete it. Because if we— one of the problems we've always been with them changing it is that they also have to go through, delete in history. Sometimes there's old stuff and whatever. I just changed the record, and then five minutes after that, it was correctly pointing to the new server, right. So.

[00:15:29] Speaker 2: Yeah, it works well too. Like, when I was looking it up earlier, like, it even has the server records for, like, aldep and everything.

[00:15:38] Speaker 1: Sorry, what?

[00:15:41] Speaker 2: Like, for the DRE, it even has, like, the server records, SRV, for, like, aldep services and stuff like that, if you're trying to hit domain controllers.

[00:15:50] Speaker 1: So if you're hitting it right now, because we don't have a reverse proxy set up, I'm not sure it's going to work. Right now, I mean, let me— let me try. It's going to, but it will say no secure because we haven't installed this certificate here on this machine yet, right. So if I do the same thing on the production one, you see, it would— it would work. And also, the Jenkins server itself is pointing to here, right. So that's an important part to understand, because Jenkins has some magic variables, and we— unless you readjust those variables, Jenkins is just going to point to, like, the actual— the VM. Do you know what I mean? Does it— does it make sense?

[00:16:44] Speaker 2: Yeah, yeah.

[00:16:46] Speaker 1: So if you're a user, you're going to be kind of like, oh, I'm going to DRE, but then it points into logs to somewhere else, or whatnot. I don't know if it makes sense. Yeah, but— so now we are ready. So there's the last bit of this is the redirect. So that change has to be in the K3S repo. So we have a merge request for this, right. So that would require a server restart because we need to install— so if we go to this one, we were now at this server here, right. At this server here we have— if I do.

[00:17:30] Speaker 4: Can we— can you do the teaching after you hit merge? I just want to make sure that we don't keep builds disabled for too long.

[00:17:40] Speaker 1: Sorry, yes. So the merge— let me— it's K3S. This merge request here, right. So have a look at it. There's four changes, but we are— I'm actually deleting this file, and I'm just pointing it. So what are we doing? We're adding this— this will install the DRE certificate, and we're pointing this server to use gcassist-prod.dre, whatever. Okay. And we're— at the same time, we're not— we're telling this server to not use this anymore. We're telling it to use the actual URL. Okay, so I'm going to merge this, and after I merge it, this will cause a redeploy to both of them, but it's okay because they're disabled, so it's fine. So right now, before we do the deploy, because it hasn't done it yet, you can see this one has the DRE secret object. This one should not. This one, yeah. Oh, it did. Maybe I've done it earlier. Oh, we've had it. Sorry. So we've already installed the actual certificate, but we didn't have the ingress set up. So this was already there. Sorry, my— because I— we did it. We just didn't have the ingress set up. So what the ingress does, if you look— if you look at those templates— so the ingress is custom for every instance because you're telling it what to— what to actually ingress. Here. So we're just telling it to ingress gcassist-prod. gcassist-prod. So this is for Jenkins, and this is for the Git mirror as well, because you can also set up the Git mirror to use this as well. Right now, by default, it's set up to use the AWS, but why not have another one, right. So yeah, so ingress is basically you're kind of telling the web server, hey, if you get— you can— you can also get requests for this specific host, right. Does this make sense?

[00:21:33] Speaker 2: It does for me. I've fought with traffic quite a bit in the past.

[00:21:37] Speaker 1: Okay. Okay, so we did this deployment, and now this— those— normally when the deployment is done, I would just go here and make sure that the pods are not airing out. So it's not— this is running for 2 minutes, not 14 days. So that means it just restarted. Same for this as well, because this one is also going to be redeployed. Okay, so now what I need to do, this one— so we can also investigate if the history— it should have been reloaded successfully. Now when I log in, there shouldn't be any errors or anything, right. So it.



## Tags

#Meeting

# Related notes

- [[2026-08-19 Debugging Jenkins login failure direct vs proxy with OIDCEntra SSO]]
- [[2026-08-19 Jenkins Active Directory plugin LDAP edge-case causing API token errors]]
- [[2026-08-19 Task list post chat, work on proposal, do shift, check on guy]]