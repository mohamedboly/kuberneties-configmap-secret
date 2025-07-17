# Gestion de la Configuration dans Kubernetes : ConfigMap et Secret

## ✨ Contexte

Lorsqu'on développe une application, on utilise souvent des informations de configuration (comme l'URL, le nom d'utilisateur et le mot de passe d'une base de données). Il est déconseillé de **les coder en dur** dans le code source.

Une bonne pratique consiste à stocker ces informations dans :

* des **variables d'environnement**
* ou des **fichiers de configuration** que l'application lit dynamiquement

Ainsi, si la configuration change, il suffit de modifier ces fichiers ou variables sans toucher au code source.

✅ Mais attention :

* Si on les place dans une classe de configuration, elle sera visible sur GitHub si le répertoire est versionné.
* On utilise donc souvent des fichiers `.env` ou `.json`, exclus du commit via `.gitignore`.

## 🚀 Kubernetes : Comment gérer ces configurations ?

Kubernetes propose deux objets :

* **ConfigMap** : pour les données non sensibles
* **Secret** : pour les données sensibles (mots de passe, tokens, etc.)

Ces objets peuvent être **injectés comme variables d'environnement ou montés comme fichiers** dans les pods.

---

## 📂 ConfigMap

### Exemple :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-cm
data:
  db-port: "3306"
```

### Commandes utiles :

```bash
kubectl apply -f test-cm.yml       # Créer la ConfigMap
kubectl get cm                      # Lister les ConfigMap
kubectl describe cm test-cm        # Détails d'une ConfigMap
```

---

### ⚡ Utiliser la ConfigMap dans un pod (via variables d'environnement)

Voici un `Deployment` avec l'injection de la variable `DB-PORT` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-python-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-python-app
  template:
    metadata:
      labels:
        app: sample-python-app
    spec:
      containers:
      - name: python-app
        image: mohamedbolydev/python-sample-app-demo:v1
        env:
        - name: DB-PORT
          valueFrom:
            configMapKeyRef:
              name: test-cm
              key: db-port
        ports:
        - containerPort: 8000
```

**Appliquer le déploiement :**

```bash
kubectl apply -f deployment.yml
```

**Tester dans le pod :**

```bash
kubectl exec -it <pod-name> -- /bin/bash
printenv | grep DB
```

---

### 🚫 Limite de cette méthode :

Si on modifie la ConfigMap, les pods **n'utiliseront pas automatiquement** la nouvelle valeur. Il faudra les redéployer.

---

### 📂 Solution : utiliser ConfigMap comme volume (fichier)

#### ✍þ Exemple de `Deployment` complet :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-python-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-python-app
  template:
    metadata:
      labels:
        app: sample-python-app
    spec:
      containers:
      - name: python-app
        image: mohamedbolydev/python-sample-app-demo:v1
        volumeMounts:
        - name: db-connection
          mountPath: /opt
        ports:
        - containerPort: 8000
      volumes:
      - name: db-connection
        configMap:
          name: test-cm
```

**Tester dans le pod :**

```bash
kubectl exec -it <pod-name> -- /bin/bash
cat /opt/db-port
```

✅ **Avantage** : toute modification de la ConfigMap est automatiquement reflétée dans les pods **sans redéploiement**.

---

## 🔒 Secret

Les `Secret` sont similaires aux `ConfigMap` mais conçus pour les données sensibles. Les valeurs sont **encodées en Base64** (ce n'est pas un vrai chiffrement).

### Exemple :

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-db-password
data:
  password: dmFsdWUtMg0KDQo  # "valeur-2" en base64
```

### Commandes utiles :

```bash
kubectl apply -f secret.yml
```

### Utilisation du Secret comme fichier monté :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-python-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-python-app
  template:
    metadata:
      labels:
        app: sample-python-app
    spec:
      containers:
      - name: python-app
        image: mohamedbolydev/python-sample-app-demo:v1
        volumeMounts:
        - name: db-password
          mountPath: /opt
        ports:
        - containerPort: 8000
      volumes:
      - name: db-password
        secret:
          secretName: secret-db-password
```

**Dans le pod :**

```bash
cat /opt/password  # Affiche "valeur-2"
```

---

## 🔎 Conclusion

* Utilisez `ConfigMap` pour les données non sensibles (ports, noms de services).
* Utilisez `Secret` pour les données sensibles (mots de passe, tokens).
* Préférez les volumes à l'injection par variable d'environnement si vous souhaitez que les modifications soient prises en compte sans redéploiement.


```
