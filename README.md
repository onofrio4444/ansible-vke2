# rke2-ansible

Automazione Ansible per creare un cluster **RKE2** single-node (CNI **canal**), con
storage **Longhorn** (CSI con snapshot, StorageClass **tanzu-sp**), **MetalLB**
per i LoadBalancer, e **Rancher** (+ cert-manager).

## Prerequisiti

- VM **RHEL** con accesso a internet.
- Sulla VM da cui lanci (bastion / control box): `ansible-core` installato.
  Non servono kubectl ne' helm: girano via SSH sul nodo.
- Accesso **SSH** verso la VM con un utente che puo' fare `sudo`.

## Come si usa

1. Sistema l'IP e l'utente SSH nell'inventario `inventory/test.ini`
   (1 nodo all-in-one).

2. Configura le variabili in `group_vars/all.yml`. Da toccare sicuramente:
   - `rke2_token`               -> stringa segreta a piacere
   - `metallb_ip_range`         -> IP liberi per i LoadBalancer
   - `rancher_hostname`         -> nome che risolve verso un IP del cluster
   - `rancher_bootstrap_password` -> password admin iniziale (>=12 caratteri)

3. Verifica la connettivita' e lancia:
   ```bash
   ansible -i inventory/test.ini all -m ping
   ansible-playbook -i inventory/test.ini site.yml
   ```

## Ordine di installazione

1. Prerequisiti OS
2. RKE2 server
3. Longhorn: storage CSI con snapshot (richiesto da Commvault per il
   backup dei volumi). Crea le StorageClass "longhorn" e "tanzu-sp"
   (stesso storage, nome atteso dai manifest applicativi) e la
   VolumeSnapshotClass "longhorn-snapclass".
4. MetalLB (+ pool IP)
5. cert-manager (HelmChart, applicazione autonoma: c'e' anche senza
   Rancher ed e' backuppabile da Commvault come namespace "cert-manager")
6. Rancher
7. Service account Commvault per i backup (a fine run stampa il token
   da inserire in Commvault; endpoint API: https://<IP-del-nodo>:6443)
8. PostgreSQL di test (StatefulSet su StorageClass tanzu-sp) con database
   "testdb" popolato di dati sintetici (clienti e ordini casuali)
9. Webapp Python (Flask) che mostra una pagina HTML con lo stato del
   collegamento a PostgreSQL e un estratto dei dati; esposta su un IP
   del pool MetalLB (l'URL viene stampato a fine run)

## Verifica (dal nodo del cluster, via SSH)

```bash
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
export PATH=$PATH:/var/lib/rancher/rke2/bin
kubectl get nodes
kubectl get storageclass
kubectl get pods -n metallb-system
kubectl get pods -n cert-manager
kubectl get pods -n cattle-system
kubectl get ingress -n cattle-system
```
Quando i pod di cattle-system sono Running, apri https://<rancher_hostname>.

## Versioni fissate (modificabili in group_vars/all.yml)

- Longhorn: ultima stabile della repo chart (longhorn_version vuoto)
- MetalLB: v0.15.2
- cert-manager: v1.20.2   (verifica la support matrix di Rancher prima di aggiornare)
- Rancher: ultima stabile della repo (rancher_version vuoto)

## Nota

RKE2 installa di suo `rke2-ingress-nginx`: Rancher lo usa per esporre la UI.
Se in futuro migri app con il proprio ingress (es. Kong), valuta col responsabile
se disabilitare quello di RKE2.
