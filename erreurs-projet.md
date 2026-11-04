# Erreurs rencontrées pendant le projet

Pour chaque erreur : l'objectif, la commande lancée, l'erreur obtenue, les logs, la cause, et comment on l'a réglée.
Empreinte du projet : `jtsr-04-11-2026`.

---

## Erreur 1 : le push GitHub est rejeté (Internal Server Error)

- **Objectif** : envoyer le dépôt local vers GitHub.
- **Commande** : `git push -u origin main`
- **Erreur obtenue** :
  ```
  remote: Internal Server Error
  ! [remote rejected] main -> main (Internal Server Error)
  error: failed to push some refs to 'https://github.com/Jonathan3731/5SDWANA_Projet.git'
  ```
  Plus tard, le message est devenu `! [remote rejected] main -> main (failure)`.
- **Logs** : erreur identique sur le tout premier push (3 petits objets) et sur le push complet (66 objets, 1,64 MiB), avec un identifiant de requête GitHub à chaque fois.
- **Cause** : l'erreur venait du serveur GitHub pour ce dépôt précis (le rejet arrive après l'envoi complet), pas de nos fichiers ni de l'authentification, qui passait.
- **Solution** : réessais, puis test en SSH, puis recréation du dépôt vide (Private, sans README, sans .gitignore, sans licence) et nouveau push. (À compléter : ce qui a fini par fonctionner chez vous.)

## Erreur 2 : SSH refusé (Permission denied publickey)

- **Objectif** : contourner l'erreur 1 en poussant en SSH au lieu de HTTPS.
- **Commande** : `git push -u origin main` après `git remote set-url origin git@github.com:...`
- **Erreur obtenue** : `git@github.com: Permission denied (publickey).` puis `fatal: Could not read from remote repository.`
- **Cause** : la clé SSH de la VM n'était pas encore enregistrée sur le compte GitHub.
- **Solution** : génération de la clé (`ssh-keygen -t ed25519`), ajout de la clé publique dans GitHub (Settings > SSH and GPG keys). Le test `ssh -T git@github.com` a répondu « successfully authenticated », mais le push restait refusé : nous sommes revenus à HTTPS avec un token.

## Erreur 3 : push Docker Hub refusé (insufficient_scope)

- **Objectif** : publier les images sur Docker Hub.
- **Commande** : `docker push jonathan3731/shop-jtsr-04-11-2026:v1`
- **Erreur obtenue** : `push access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed`
- **Cause** : l'identifiant utilisé dans le nom de l'image (`jonathan3731`, celui de GitHub) n'était pas l'identifiant Docker Hub (`jonathan57`). Docker Hub n'autorise l'écriture que dans l'espace de son propre compte.
- **Solution** : `docker login` avec le bon compte, puis renommage des images avec `docker tag` vers `jonathan57/...`, mise à jour du nom des images dans les manifests, puis `docker push`.

## Erreur 4 : gunicorn ne peut pas écrire (Permission denied /.gunicorn)

- **Objectif** : lancer l'image `api` en utilisateur non root.
- **Commande** : `docker run -d --name api-jtsr-04-11-2026 ...` puis `docker logs api-jtsr-04-11-2026`
- **Erreur obtenue** : `[ERROR] Control server error: [Errno 13] Permission denied: '/.gunicorn'`
- **Cause** : gunicorn 26 crée un dossier `.gunicorn` dans le dossier personnel de l'utilisateur. L'utilisateur `nobody` a `/` comme dossier personnel, où il ne peut pas écrire.
- **Solution** : ajout de `ENV HOME=/tmp` avant `USER nobody` dans les Dockerfile de `api` et `shop`, puis rebuild. Après correction, plus aucune ligne `[ERROR]` dans les logs.

## Erreur 5 : `sed` ne trouve pas les manifests

- **Objectif** : remplacer l'ancien identifiant Docker Hub dans les manifests.
- **Commande** : `sed -i "s/jonathan3731/jonathan57/g" k8s/*.yaml`
- **Erreur obtenue** : `sed: can't read k8s/*.yaml: No such file or directory`
- **Cause** : les manifests n'étaient pas encore créés, le dossier `k8s/` était vide.
- **Solution** : créer d'abord les fichiers avec `nano`, en écrivant directement `jonathan57` et l'empreinte `jtsr-04-11-2026`.

## Erreur 6 : Stripe refuse la clé (Invalid API Key provided)

- **Objectif** : lancer un paiement test depuis la boutique.
- **Commande** : clic sur le bouton Checkout de `http://172.18.75.182:8081`
- **Erreur obtenue** : `{"error":"Invalid API Key provided: sk_test_..."}` sur la page `/create-checkout-session`.
- **Logs** : `kubectl logs -n mlops-jtsr-04-11-2026 -l app=shop-jtsr-04-11-2026`
- **Cause** : le Secret Kubernetes contenait une valeur d'exemple au lieu de la vraie clé de test Stripe.
- **Solution** : copie des vraies clés depuis le dashboard Stripe (mode Test), correction du Secret avec `kubectl apply -f secret.env`, puis `kubectl rollout restart deployment shop-jtsr-04-11-2026`, car les variables d'un Secret ne sont lues qu'au démarrage des pods.

## Erreur 7 : Netdata inaccessible (ERR_CONNECTION_REFUSED)

- **Objectif** : ouvrir l'interface Netdata depuis le PC.
- **Commande** : navigateur sur `http://172.18.75.182:19999`
- **Erreur obtenue** : `Ce site est inaccessible, ERR_CONNECTION_REFUSED`
- **Cause** : le port-forward vers Netdata n'était plus actif (il s'arrête avec le terminal ou le pod).
- **Solution** : relance de `kubectl port-forward --address 0.0.0.0 -n monitoring-jtsr-04-11-2026 svc/netdata-jtsr-04-11-2026 19999:19999 &`. L'option `--address 0.0.0.0` est nécessaire pour que le PC puisse se connecter, pas seulement la VM.

---

## Ce qu'on a retenu

- Toujours vérifier les identifiants (GitHub, Docker Hub, Stripe) avant de chercher un bug dans le code.
- Lire les logs (`docker logs`, `kubectl logs`, `kubectl describe`) : l'erreur y est presque toujours écrite clairement.
- Un Secret ou un ConfigMap modifié n'est pas pris en compte tant que les pods ne sont pas redémarrés.
