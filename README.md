# OFA-TASKS
Gestion des tâches planifiés - nouvel espace de Batch sous Symfony
 
#Environnement de developpement (sur votre poste de travail)

### Requirements :
 * les variables d'environnements DOCKER_PATH_ENV, UID et GID doit etre correctement definies dans le fichier ~/.bashrc
      - export DOCKER_PATH_ENV=/home/nicolas (à adapter)
      - export COMPOSE_UID=$UID
      - export COMPOSE_GID=$GID
 * ce repo doit etre cloné dans ${DOCKER_PATH_ENV}/data/ofa-tasks
 * les chemins suivant doivent exister
      - ${DOCKER_PATH_ENV}/data/ofa-tasks
      - ${DOCKER_PATH_ENV}/data/ofa-data
      - ${DOCKER_PATH_ENV}/data/ofa-common
 * vous devez creer en local un fichier .env.dev en vous basant sur .env.build
   attention a ne pas sauvegarder ce fichier dans git
 * Si vous avez deja lance ofa-task il faut supprimer le volume : 
    `docker-compose down && docker volume rm ofatasks_vendorofatasks`
 * Créer le dossier vendor à la racine du projet

### Build & run
 * si les requirements sont ok : docker-compose build
 * pour lancer l'environnement : docker-compose up
 * Pour lancer RabbitMQ : `docker-compose -f docker-compose.yml -f docker-compose.rabbitmq.yml up -d`
 * pour se rendre dans le container php : `docker exec -ti php-app-tasks bash` 

si vous voulez juste lancer l'image sans passer par le docker-compose : `docker run --rm -v$(pwd)/.env.dev:/data/www/ofa-tasks/.env  ofa-tasks php bin/console app:test`
l'idée est de monter le .env que vous souhaitez ...

## Lancer une tâche en dev

### Se connecter à la vm docker
```
docker exec -it php-app-tasks bash
```
### Commande
```
php bin/console app:push-mongodb-vitrine
```
### Commande avec des traces
```
php bin/console app:push-mongodb-vitrine -vvv
```
### Depiler des messages avec traces
```
php bin/console messenger:consume nom_de_la_queue -vvv
```


JENKINS
============
=> https://jenkins.of2m.fr/job/BU-Auto/job/10-OFA-Tasks/

2 pipelines :
- le 1er nommé "ofa-tasks" est chargé de rebuilder l'image eu.gcr.io/auto-151812/ofa-tasks avec comme tag master.BUILD_NUMBER
- le 2eme pipeline paramétré fait 2 choses : il vous demande un BUILD_NUMBER et va donner le tag prod à l'image en question, il va également mettre a jour la configmap kubernetes nommée "symfonyenv" a partir du secrefile jenkins nommé "ofa-tasks---synfony_env_prod"

le secret jenkins se gere via l'url suivante : https://jenkins.of2m.fr/job/BU-Auto/credentials/store/folder/domain/_/credential/ofa-tasks---synfony_env_prod/
vous ne pouvez pas le voir, vous pouvez le supprimer et le recreer (pour le modifier)
ce secretfile contient le fichier .env de symfony


pour voir ce que contient la configmap "symfonyenv" et par assiocation le secret Jenkins nommé "ofa-tasks---synfony_env_prod", il suffit d'interroger kubernetes.

    kubectl -n ofa-tasks-prod get configmap symfonyenv  --output 'jsonpath={.data.\.env}'



KUBERNETES
=============
tout se passe, sur le cluster ofa-cluster01
j'ai créé un namespace nommé "ofa-tasks-prod"
dans ce namespace, on retrouve la configmap nommée "symfonyenv" qui va etre monté dans /data/www/ofa-tasks/.env de les containers.


l'equivalent du cron unix que l'on connait est l'object cronjob dans kubernetes : vous trouvez un exemple dans le repertoire cloud/configProd
pour creer un autre cronjob ? c'est simple, je copie le fichier d'exemple et je le modifie :
- je change tous les `"name: apptest"` par le nom de que veux donner à mon job
- je change le schedule en fonction de mes besoins **/!\ le schedule de l'heure ce fait en UTC±00:00**
- je change la commande a lancer => command: `["php", "bin/console", "app:test"]`

    je n'ai plus qu'a appliquer mon nouveau yaml : `kubectl apply -f new_cronjob.yaml -n ofa-tasks-prod`

---

* pour voir la liste des cronjobs: `kubectl -n ofa-tasks-prod get cronjobs.batch`
* pour supprimer un cronjob : `kubectl -n ofa-tasks-prod delete cronjobs.batch cronjob_name`
* pour voir les jobs deja lancés ou en cours : `kubectl  -n ofa-tasks-prod get pods`
* pour lancer un cronjob manuellement : `kubectl -n ofa-tasks-prod  create job --from=cronjob/appalertclient appalertclient-test`

### RabbitMQ recette

* Pour mettre en place le RabbitMQ de rectte pour effectuer des test en prod sans utiliser le RabbitMQ de prod, il suffit de se placer dans le dossier ofa-tasks/cloud/configProd et lancer la commande: `kubectl apply -f rabbitmq-recette.yaml`.


* Une fois les test terminer il faut supprimer le pod RabbitMQ recette en lancant la commande: `kubectl delete -f rabbitmq-recette.yaml`
