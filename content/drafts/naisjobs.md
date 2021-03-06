Cronjobs/jobs på NAIS
=====================

Vi har en litt mer hands-on tilnærming når det gjelder jobs og cronjobs på Kubernetes.

Kom i gang med å følge stegene nedenfor:


## 1. Oppsett av Vault

Dette er for å kunne legge til secrets. Du kan fint hoppe over dette steget hvis du ikke trenger det. Mer dokumentasjon om Vault finner du i [vault-iac](https://github.com/navikt/vault-iac/blob/master/doc/endusers.md)-repoet.


### Legg til i github.com/navikt/vault-iac/terraform/${cluster}.tf

```
module "${team_name}-${cluster}" {
  application_name = "${team_name}"
  namespaces = [ "${namespace}" ]
  source = "./modules/naisapp"
  zone = "fss"
  environment = "prod"
}
```


### Legg til i github.com/navikt/vault-iac/terraform/ldap.tf

Det er viktig at `${cluster}` i denne sammenhengen skrives med `_`/understrek og ikke `-`/bindestrek.
```
module "${team_name}" {
  ldap_group = "${ldap_group}" 
  policies = [
    "${cluster}_${team_name}_sudo",
  ]
  source = "./modules/ldap"
}
```


## 2. Eget namespace

Vi tillater kun cronjobs/jobs i egne namespaces. Hvis `vars/<cluster>/naisjobs.yaml` ikke finnes i cluster-katalogen må du opprette den. Da må du samtidig kopiere `naisjobs.yaml` til tilsvarende `templates/<cluster>`-katalog.


### Legg til github.com/navikt/nais-yaml/vars/${cluster}/jobs.yaml

```
naisjobs:
- name: ${namespace}
  group_name: ${teamname}
  group_id: ${ldap_group_id}
```

Finnes filen fra før av, legger du bare til informasjon på siste linje.


## 3. Maskinbruker for å kunne opprette cronjobs/jobs

Man vil kunne lage cronjobs/jobs med sin egen bruker, men trenger man en maskinbruker kan man få det fra oss/NAIS. Spør i [#nais](https://nav-it.slack.com/messages/C5KUST8N6) eller ta direkte kontakt med [@kyrre.havik.eriksen](https://nav-it.slack.com/messages/D8QQ9ELK1).

Det vi trenger å vite er `cluster` og `team`.


## 4. kubectl apply -f job.yml

Når alle de andre stegene er gjort, så kan dere bruke deres egen bruker, eller maskinbrukeren dere fikk i steg 3 til å `apply`e en yaml-fil i clusteret!

Nedenfor har dere et godt utgangspunkt for hvordan en slik yaml-fil kan se ut.
cronjob.yml/job.yml
```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: ${jobname}
  namespace: ${namespace}
  labels:
    team: ${teamname}
spec:
  schedule: ${schedule}
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: ${jobname}
            image: repo.adeo.no:5443/<app-name>:<version>
            volumeMounts:
            - mountPath: /etc/ssl/certs/ca-certificates.crt
              name: ca-bundle
              subPath: ca-bundle.pem
            - mountPath: /etc/pki/tls/certs/ca-bundle.crt
              name: ca-bundle
              subPath: ca-bundle.pem
            - mountPath: /etc/ssl/ca-bundle.pem
              name: ca-bundle
              subPath: ca-bundle.pem
            - mountPath: /etc/pki/tls/cacert.pem
              name: ca-bundle
              subPath: ca-bundle.pem
            - mountPath: /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem
              name: ca-bundle
              subPath: ca-bundle.pem
            - mountPath: /etc/ssl/certs/java/cacerts
              name: ca-bundle
              subPath: ca-bundle.jks
            - mountPath: /var/run/secrets/nais.io/vault
              name: vault-secrets
            resources:
              requests:
                memory: "128Mi"
                cpu: "250m"
              limits:
                memory: "256Mi"
                cpu: "500m"
          initContainers:
          - env:
            - name: VKS_VAULT_ADDR
              value: https://vault.adeo.no
            - name: VKS_AUTH_PATH
              value: ${vks_auth_path}
            - name: VKS_KV_PATH
              value: ${vks_kv_path}
            - name: VKS_VAULT_ROLE
              value: ${team_name}
            - name: VKS_SECRET_DEST_PATH
              value: /var/run/secrets/nais.io/vault
            image: navikt/vks:29
            name: vks
            volumeMounts:
            - mountPath: /var/run/secrets/nais.io/vault
              name: vault-secrets
          serviceAccount: podcreator
          serviceAccountName: podcreator
          volumes:
          - configMap:
              defaultMode: 420
              name: ca-bundle
            name: ca-bundle
          - emptyDir:
              medium: Memory
            name: vault-secrets
```

Man kan fjerne `spec.jobTemplate.spec.template.spec.initContainers`-innslaget hvis man ikke trenger Vault/hemmeligheter.
