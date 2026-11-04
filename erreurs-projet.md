## Erreur : gunicorn Permission denied /.gunicorn
- Objectif : lancer l'image api en utilisateur non root
- Commande : docker run -d --name api-jtsr-04-11-2026 ...
- Erreur obtenue : Control server error: [Errno 13] Permission denied: '/.gunicorn'
- Solution : ajout de ENV HOME=/tmp dans le Dockerfile (le dossier personnel de nobody est / et n'est pas inscriptible)
