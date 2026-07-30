# Jenkins_jdk25
A home lab without DNS/AD actually simplifies things — you skip LDAP entirely (local Jenkins auth is the right call anyway) and just need to reach Jenkins by IP:port or an /etc/hosts entry instead of a real domain. I'll give you one clean, consolidated set of files.

One honest limit up front: I don't have access to your home lab cluster from this sandbox — I can't run kubectl apply against your actual nodes. What I can do is validate every YAML file's syntax here (real output below), give you the exact command sequence with what output to expect, and you run it on your machine. Let's go.

Architected authentication strategy and deployment configuration for homelab Jenkins
Architected authentication strategy and deployment configuration for homelab Jenkins

Now let's actually validate all of these — real command, real output:

Validate all homelab manifest YAML files
Validate all homelab manifest YAML files

All six files parse correctly and contain the expected Kubernetes object kinds. Now let's package them for you:

Ran a command, read a file
Ran a command, read a file
Steps to run on your home lab machine

1. Edit two things first:

01-nfs-pv-pvc.yaml → set spec.nfs.server and spec.nfs.path to your real NFS box
03-configmaps.yaml → the unclassified.location.url at the bottom — leave the placeholder for now, you'll fix it after step 6 once you know your actual node IP

2. Confirm kubectl is pointed at your home lab cluster

bash
kubectl cluster-info
kubectl get nodes

3. Create the namespace

bash
kubectl apply -f 00-namespace.yaml

4. Create the admin credentials secret (don't hardcode this in a file)

bash
kubectl -n jenkins create secret generic jenkins-admin-credentials \
  --from-literal=username=admin \
  --from-literal=password="$(openssl rand -base64 18)"

Save that password now — you'll need it to log in:

bash
kubectl -n jenkins get secret jenkins-admin-credentials -o jsonpath='{.data.password}' | base64 -d; echo

5. Apply everything else in order

bash
kubectl apply -f 01-nfs-pv-pvc.yaml
kubectl apply -f 02-serviceaccount-rbac.yaml
kubectl apply -f 03-configmaps.yaml
kubectl apply -f 04-deployment.yaml
kubectl apply -f 05-service.yaml

6. Watch it come up

bash
kubectl -n jenkins get pods -w

Expect: install-plugins init container runs first (takes a minute or two downloading plugins), then the jenkins container starts. Ctrl+C once you see 1/1 Running.

7. Find your access URL — no DNS needed

bash
kubectl get nodes -o wide

Take any node's INTERNAL-IP (e.g. 192.168.1.100) and browse to:

http://192.168.1.100:30080

NodePort 30080 is fixed in the Service manifest, so this URL won't change across redeploys.

8. Optional — give it a friendly name without real DNS
On the machine(s) you browse from, add to /etc/hosts (or C:\Windows\System32\drivers\etc\hosts on Windows):

192.168.1.100  jenkins.home

Then update unclassified.location.url in 03-configmaps.yaml to http://jenkins.home:30080/, reapply, and restart the pod — this avoids the reverse-proxy URL mismatch we talked about earlier, since Jenkins' internal idea of its own URL will now match what's in your browser.

9. Log in at that URL with username admin and the password from step 4.

If any kubectl apply step errors or a pod doesn't reach Running, paste the output here and I'll help you debug it directly.

--------------------------------------

python3 -c "import yaml; yaml.safe_load(open('04-deployment.yaml'))" && echo "YAML OK"
grep -c "secretKeyRef" 04-deployment.yaml    # should print 2
grep "runAsNonRoot" 04-deployment.yaml       # should print the line, not runAsRoot

kubectl -n jenkins delete pod jenkins-848bb974b8-2zbt5
kubectl apply -f 04-deployment.yaml
kubectl -n jenkins get pods -w

kubectl -n jenkins logs -f jenkins-<new-pod-suffix> -c install-plugins

kubectl apply -f 04-deployment.yaml
kubectl -n jenkins delete pod -l app=jenkins
kubectl -n jenkins get pods -w
kubectl -n jenkins logs -f -l app=jenkins -c install-plugins

------------------------------------
Since your admin credentials come from a Kubernetes Secret via JCasC (not stored independently in Jenkins), the right way to change it is to update the Secret and let Jenkins re-apply config on restart — not through the Jenkins UI's own password change screen, since JCasC re-provisions the local user from that Secret every time the pod starts and would silently overwrite a UI-made change on the next restart.

1
Generate a new password
Pick a strong password or generate one: openssl rand -base64 18. Save it somewhere safe (password manager) before continuing — you'll need it to log back in.
2
Update the Kubernetes Secret
kubectl -n jenkins create secret generic jenkins-admin-credentials --from-literal=username=admin --from-literal=password='<your-new-password>' --dry-run=client -o yaml | kubectl apply -f - The --dry-run=client -o yaml piped into apply updates the existing Secret in place rather than erroring that it already exists.
3
Restart the Jenkins pod
kubectl -n jenkins rollout restart deployment jenkins This triggers a fresh pod, which re-reads the Secret via the ADMIN_PASSWORD env var and JCasC reapplies it to the local security realm on startup.
4
Watch it come back up
kubectl -n jenkins get pods -w Wait for 1/1 Running before trying to log in — same startup sequence as before (init container installs plugins, then Jenkins boots and applies casc.yaml).
5
Log in with the new password
Browse to your Jenkins URL, username admin, and the new password from step 1. If login fails, double check the Secret actually updated: kubectl -n jenkins get secret jenkins-admin-credentials -o jsonpath='{.data.password}' | base64 -d

One important edge case: if you ever lose access to both the Jenkins UI and the ability to read the Secret (e.g. lost kubectl access too), the fallback is to exec into the pod and reset it via Jenkins' Groovy script mechanism directly on disk:

bash
kubectl -n jenkins exec -it deploy/jenkins -- bash

Then inside the pod, drop a Groovy init script that resets the admin password directly against Jenkins' internal user store — but that's a last-resort path since it works around JCasC rather than through it. For your normal workflow, updating the Secret + restart is the clean way, since it keeps the Secret as the single source of truth that matches what's actually running.
