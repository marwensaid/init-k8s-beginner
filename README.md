# init-k8s-beginner

> Hypothèses d’environnement (ne pas demander) : les étudiants utiliseront **minikube** (assez simple pour un TD local). Si tu préfères `kind` / `k3s`, dis-le et j’adapte. Ici j’utilise minikube dans les exemples.

---

# Objectifs du TD

1. Comprendre les concepts de base : Pod, Deployment, Service, ConfigMap, Secret, Volume, Namespace, RBAC, Probes.
2. Savoir créer et manipuler des objets Kubernetes via YAML et `kubectl`.
3. Savoir déployer une application simple (NGINX + app HTTP) et la mettre à l’échelle, mettre à jour (rolling update) et diagnostiquer.
4. S’exercer à la gestion des configurations et des données persistantes.
5. Lire l’état d’un cluster et diagnostiquer des incidents simples.

---

# Plan du TD (durée estimée — indicatif)

1. Préparation et démarrage (15 min)
2. Pods & Deployments (30 min)
3. Services (20 min)
4. ConfigMap & Secret (15 min)
5. Volumes & PVC (20 min)
6. Probes / Health checks (15 min)
7. Namespaces & RBAC & quotas (20 min)
8. Rolling update & troubleshooting (15 min)
9. Quiz / révisions (15 min)

---

# Pré-requis & setup

* Installer : `kubectl` (version compatible), `minikube`.
* Démarrer minikube :

```bash
minikube start --memory=4096 --cpus=2
# vérifier
kubectl version --client
kubectl cluster-info
kubectl get nodes
```

* Si besoin, ouvrir un tunnel (pour NodePort/LoadBalancer test) :

```bash
minikube tunnel   # (nécessite sudo, laisse tourner pour LoadBalancer)
```

---

# Notions / Définitions rapides (pour révision)

* **Node** : machine (VM/physique) qui exécute les Pods.
* **Pod** : plus petite unité déployable ; peut contenir 1+ containers partageant réseau/stockage.
* **Deployment** : objet qui gère le cycle de vie de ReplicaSets / Pods (déploiement, mise à l’échelle, rolling updates).
* **Service** : abstraction réseau pour exposer Pods (ClusterIP, NodePort, LoadBalancer).
* **ConfigMap** : stocke des données de configuration non sensibles.
* **Secret** : stocke des données sensibles (mots de passe, clés).
* **PersistentVolume (PV)** & **PersistentVolumeClaim (PVC)** : abstraction pour volumes persistants.
* **Probe** : health checks — liveness (is container alive?) / readiness (ready for traffic?).
* **Namespace** : isolation logique de ressources.
* **RBAC** : contrôle d’accès basé sur rôles (Role / ClusterRole / RoleBinding).
* **kubectl** : CLI principale pour interagir avec le cluster.

---

# Exercice 1 — Pod simple & debug

But : lancer un Pod `busybox`, se connecter dedans, observer logs.

Manifest `pod-busybox.yaml` :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-sleep
  labels:
    app: busybox
spec:
  containers:
  - name: busybox
    image: busybox:1.34
    command: ["sleep", "3600"]
```

Commandes :

```bash
kubectl apply -f pod-busybox.yaml
kubectl get pods -o wide
kubectl describe pod busybox-sleep
kubectl exec -it busybox-sleep -- sh
# inside: ps aux; exit
kubectl delete pod busybox-sleep
```

Points d’apprentissage :

* `kubectl describe` pour events.
* `kubectl exec` pour debug.

---

# Exercice 2 — Deployment + scaling

But : déployer une app HTTP (ici `hashicorp/http-echo` ou `nginx`) et scaler.

Deployment `deployment-echo.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - name: http-echo
        image: hashicorp/http-echo:0.2.3
        args:
        - "-text=Bonjour depuis pod"
        ports:
        - containerPort: 5678
```

Commands:

```bash
kubectl apply -f deployment-echo.yaml
kubectl get deployments
kubectl get pods -l app=echo
kubectl scale deployment echo-deployment --replicas=4
kubectl get pods
# rollback / update replicas
kubectl rollout status deployment/echo-deployment
```

Exercice : modifier `replicas` dans le YAML puis `kubectl apply -f` ; observer.

---

# Exercice 3 — Services : ClusterIP / NodePort / LoadBalancer

But : exposer le déploiement.

Service ClusterIP (par défaut, interne) `svc-clusterip.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: echo-clusterip
spec:
  selector:
    app: echo
  ports:
    - port: 80
      targetPort: 5678
      protocol: TCP
  type: ClusterIP
```

Service NodePort `svc-nodeport.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: echo-nodeport
spec:
  selector:
    app: echo
  type: NodePort
  ports:
    - port: 80
      targetPort: 5678
      nodePort: 30080   # optionnel : minikube autorise plage 30000-32767
```

Service LoadBalancer (minikube: nécessite tunnel) `svc-lb.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: echo-lb
spec:
  selector:
    app: echo
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 5678
```

Commandes :

```bash
kubectl apply -f svc-clusterip.yaml
kubectl get svc
kubectl apply -f svc-nodeport.yaml
kubectl get svc
# appeler depuis le cluster (pod busybox)
kubectl run -it --rm curlpod --image=radial/busyboxplus:curl --restart=Never -- sh -c "curl http://echo-clusterip"
# depuis l'extérieur pour NodePort
minikube ip   # -> IP
curl http://$(minikube ip):30080
# pour LoadBalancer (if using tunnel)
minikube tunnel &   # puis kubectl get svc echo-lb
```

Notion : différence ClusterIP (interne), NodePort (expose sur Node), LoadBalancer (cloud LB).

---

# Exercice 4 — ConfigMap & Secret

But : injecter configuration dans une app.

Créer ConfigMap:

```bash
kubectl create configmap example-config --from-literal=welcome="Bienvenue M2" --from-literal=mode="dev"
kubectl get configmap example-config -o yaml
```

Créer Secret (stringData rapide) :

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: example-secret
type: Opaque
stringData:
  DB_USER: admin
  DB_PASS: s3cr3t
```

Utiliser ConfigMap & Secret dans un Pod (snippet) :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: echo-config
spec:
  containers:
  - name: echo
    image: hashicorp/http-echo:0.2.3
    args: ["-text=$(WELCOME) $(DB_USER)"]
    env:
    - name: WELCOME
      valueFrom:
        configMapKeyRef:
          name: example-config
          key: welcome
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: example-secret
          key: DB_USER
    ports:
    - containerPort: 5678
```

Commands:

```bash
kubectl apply -f secret.yaml
kubectl apply -f (pod-using-configmap-secret).yaml
kubectl logs pod/echo-config
kubectl describe pod echo-config
kubectl delete pod echo-config
```

Point : différences stockage (base64 pour Secret), bonnes pratiques (ne pas commit en clair).

---

# Exercice 5 — Volumes & PVC (stockage persistant)

But : démontrer PVC -> PV. En minikube on peut utiliser `hostPath` (ou storageClass minikube).

PV (exemple simple - hostPath pour TD) `pv-hostpath.yaml` :

> Note : hostPath = uniquement pour démo locale.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-demo
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /tmp/data-demo   # sur node minikube
  persistentVolumeReclaimPolicy: Retain
```

PVC `pvc-claim.yaml` :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-demo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

Pod qui monte le PVC `pod-pvc.yaml` :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-pvc
spec:
  containers:
  - name: web
    image: nginx
    volumeMounts:
      - mountPath: /usr/share/nginx/html
        name: demo-vol
  volumes:
    - name: demo-vol
      persistentVolumeClaim:
        claimName: pvc-demo
```

Commandes :

```bash
kubectl apply -f pv-hostpath.yaml
kubectl apply -f pvc-claim.yaml
kubectl get pv,pvc
kubectl apply -f pod-pvc.yaml
kubectl exec -it pod-with-pvc -- ls /usr/share/nginx/html
```

Exercice : écrire une page HTML dans le volume et vérifier qu’elle est servie par NGINX.

---

# Exercice 6 — Liveness & Readiness probes

But : éviter que le service envoie du trafic à un conteneur non prêt et redémarrer si bloqué.

Deployment avec probes :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: probe-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: probe
  template:
    metadata:
      labels:
        app: probe
    spec:
      containers:
      - name: http-echo
        image: hashicorp/http-echo:0.2.3
        args: ["-text=ok","-listen=:8080"]
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 2
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
```

Tests :

* Simuler indisponibilité (modifier container pour répondre 500) — ou utiliser `kubectl exec` pour mettre le process en pause ; observer `kubectl describe pod` et `kubectl get pods` pour voir restart.

---

# Exercice 7 — Namespaces, ResourceQuota, RBAC minimal

But : créer un namespace, appliquer quota, créer un Role et RoleBinding.

Namespace :

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: td-namespace
```

ResourceQuota simple :

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: rq-small
  namespace: td-namespace
spec:
  hard:
    pods: "5"
    requests.cpu: "1"
    requests.memory: 1Gi
```

Role + RoleBinding (autoriser list/get pods dans ns) :

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: td-namespace
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: td-namespace
subjects:
- kind: User
  name: alice   # pour TD on peut utiliser ServiceAccount plutôt
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Astuce pour TD : plutôt créer un `ServiceAccount` et binder à cette SA pour démonstration :

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: td-reader
  namespace: td-namespace
```

Puis RoleBinding with subject kind ServiceAccount.

Commandes :

```bash
kubectl apply -f namespace.yaml
kubectl apply -f resourcequota.yaml
kubectl apply -f serviceaccount-role-rolebinding.yaml
kubectl auth can-i list pods --as=system:serviceaccount:td-namespace:td-reader -n td-namespace
```

---

# Exercice 8 — Rolling update & rollback (Deployment)

But : montrer changement d’image et rollback.

1. Déployer nginx `1.19`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
        ports:
        - containerPort: 80
```

2. Mettre à jour image `nginx:1.21`. Puis :

```bash
kubectl apply -f nginx-deploy.yaml   # après modification image
kubectl rollout status deployment/nginx-deploy
kubectl rollout history deployment/nginx-deploy
# rollback
kubectl rollout undo deployment/nginx-deploy
```

Observer les ReplicaSets (`kubectl get rs`) pour comprendre.

---

# Diagnostic & commandes utiles (cheat-sheet)

* `kubectl get pods -A` : tous les pods dans tous les namespaces.
* `kubectl describe pod <pod>` : events & détails.
* `kubectl logs <pod> [-c container]` : logs.
* `kubectl exec -it pod -- sh` : shell dans le pod.
* `kubectl port-forward svc/my-svc 8080:80` : accéder localement au service.
* `kubectl apply -f file.yaml` / `kubectl delete -f file.yaml`
* `kubectl get events --sort-by=.metadata.creationTimestamp`
* `kubectl top pod` / `kubectl top node` (si metrics-server installé).
* `kubectl explain deployment` — pour documentation.
* `kubectl cluster-info dump` — diagnostique cluster.

---

# Cas pratiques / Challenges (à donner aux groupes)

1. **Challenge A (réseau)** : Déployer 2 apps (A et B). A doit appeler B via service DNS `http://b-service`. Montrer le mécanisme de découverte DNS. Ajouter un `Circuit Breaker` simulé (timeout côté A).
2. **Challenge B (config & secret)** : Déployer une app qui lit un `DATABASE_URL` depuis Secret; chiffrer le secret en base64 à la main et déployer; démontrer `kubectl get secret -o yaml` et comment le récupérer en clair (`kubectl get secret -o go-template=...`).
3. **Challenge C (persistance)** : Déployer un petit blog (ex: ghost si disponible) et rendre la DB persistante via PVC. Expliquer les limites de `hostPath`.
4. **Challenge D (sécurité)** : Créer un Role sans permission de supprimer pods et montrer `kubectl auth can-i delete pods --as=...`.

---

# Solutions / exemples attendus (résumés)

Pour chaque exercice ci-dessus, fournir les manifests fournis plus haut comme solution. Expliquer les outputs typiques :

* `kubectl get pods` → STATUS: Running / CrashLoopBackOff / Pending.
* `kubectl describe pod` → events montrant `FailedScheduling` (si quota/ressources), `ImagePullBackOff`, etc.
* `kubectl logs` → erreurs d’app.

---

# Quiz & Questions de révision

1. Qu’est-ce qu’un Pod ?
   Réponse : unité atomique qui contient 1+ containers partageant réseau et volumes.

2. Différence Deployment vs ReplicaSet ?
   Réponse : Deployment gère ReplicaSets ; on manipule Deployments (rolling update, rollback).

3. Quels types de Service existe-t-il et quand utiliser chacun ?
   Réponse : ClusterIP (interne), NodePort (expose port sur node), LoadBalancer (cloud LB), ExternalName (DNS CNAME).

4. Quand utiliser ConfigMap vs Secret ?
   Réponse : ConfigMap pour config non sensible; Secret pour données sensibles (stockage encodé en base64).

5. Que fait une readinessProbe ?
   Réponse : indique si pod est prêt à recevoir du trafic ; si non, Service ne route pas vers lui.

6. Que signifie CrashLoopBackOff ?
   Réponse : container démarre puis crash en boucle ; souvent erreur d’exécution / Args / image.

---

# Annexes utiles (commands & snippets)

* Voir logs d’un ReplicaSet en update :

```bash
kubectl rollout status deployment/echo-deployment
kubectl rollout history deployment/echo-deployment
kubectl describe rs <replicaset-name>
```

* Supprimer un namespace (attention : supprime tout dedans) :

```bash
kubectl delete namespace td-namespace
```

---

