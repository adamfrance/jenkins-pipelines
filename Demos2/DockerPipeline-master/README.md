!["Docker Pulls](https://img.shields.io/docker/pulls/hakdogan/jenkins-pipeline.svg)
[![Analytics](https://ga-beacon.appspot.com/UA-110069051-1/jenkins-pipeline/readme)](https://github.com/igrigorik/ga-beacon)

# Un tutoriel sur l'integration continue et le deploiement continue en utilisant Docker dans un Pipeline Jenkins 

Ce référentiel est un didacticiel qui tente d'illustrer comment gérer automatiquement le processus de création, de test avec la couverture la plus élevée et les phases de déploiement.

Notre objectif est de nous assurer que notre pipeline fonctionne bien après chaque envoi de code. 
Les processus que nous voulons gérer automatiquement : 
* Verifier le code 
* Exécuter des tests 
* Compiler le code 
* Exécuter l'analyse Sonarqube sur le code 
* Créer une image Docker 
* Pousser l'image vers Docker Hub 
* Extraire et exécuter l'image

## Premiere etape, lancer les services

Puisque l'un des objectifs est d'obtenir le rapport ''sonarqube'' de notre projet, nous devrions pouvoir accéder à sonarqube depuis le service jenkins. ''Docker compose'' est le meilleur choix pour exécuter des services travaillant ensemble. Nous configurons nos services applicatifs dans un fichier yaml comme ci-dessous.

``docker-compose.yml``
```yml
version: '3.2'
services:
  sonarqube:
    build:
      context: sonarqube/
    ports:
      - 9000:9000
      - 9092:9092
    container_name: sonarqube
  jenkins:
    build:
      context: jenkins/
    privileged: true
    user: root
    ports:
      - 8080:8080
      - 50000:50000
    container_name: jenkins
    volumes:
      - /tmp/jenkins:/var/jenkins_home #N'oubliez pas que le répertoire tmp est conçu pour être effacé au redémarrage du système.
      - /var/run/docker.sock:/var/run/docker.sock
    depends_on:
      - sonarqube
```

Les chemins d'accès des fichiers docker des conteneurs sont spécifiés dans l'attribut context dans le fichier docker-compose. Contenu de ces fichiers comme suit.

``sonarqube/Dockerfile``
```
FROM sonarqube:6.7-alpine
```

``jenkins/Dockerfile``
```
FROM jenkins:2.60.3
```

Si nous exécutons la commande suivante dans le même répertoire que le fichier ''docker-compose.yml'', les conteneurs Sonarqube et Jenkins seront opérationnels.
```
docker-compose -f docker-compose.yml up --build
```

```
docker ps

CONTAINER ID        IMAGE                COMMAND                  CREATED              STATUS              PORTS                                              NAMES
87105432d655        pipeline_jenkins     "/bin/tini -- /usr..."   About a minute ago   Up About a minute   0.0.0.0:8080->8080/tcp, 0.0.0.0:50000->50000/tcp   jenkins
f5bed5ba3266        pipeline_sonarqube   "./bin/run.sh"           About a minute ago   Up About a minute   0.0.0.0:9000->9000/tcp, 0.0.0.0:9092->9092/tcp     sonarqube
```

## GitHub configuration
Nous allons définir un service sur Github pour appeler le ''Jenkins Github webhook'' car nous voulons déclencher le pipeline. Pour ce faire, allez à _Settings -> Intégrations et services._ Le ''plugin Jenkins Github'' devrait être affiché dans la liste des services disponibles comme ci-dessous.

![](images/001.png)

Après cela, nous devrions ajouter un nouveau service en tapant l'URL du conteneur Jenkins dockerisé avec le chemin ''/github-webhook/''.

![](images/002.png)

L'étape suivante consiste à créer une ''clé SSH'' pour un utilisateur Jenkins et à la définir comme ''Déployer des clés'' sur notre référentiel GitHub.

![](images/003.png)

Si tout se passe bien, la demande de connexion suivante devrait revenir avec succès.
```
ssh git@github.com
PTY allocation request failed on channel 0
Hi <your github username>/<repository name>! You've successfully authenticated, but GitHub does not provide shell access.
Connection to github.com closed.
```

## Jenkins configuration

Nous avons configuré Jenkins dans le fichier de composition docker pour fonctionner sur le port 8080, donc si nous visitons http://localhost:8080 nous serons accueillis avec un écran comme celui-ci.

![](images/004.png)

Nous avons besoin du mot de passe administrateur pour procéder à l'installation. Il est stocké dans le répertoire ''/var/jenkins_home/secrets/initialAdminPassword'' et est également écrit en sortie sur la console lorsque Jenkins démarre.

```
jenkins      | *************************************************************
jenkins      |
jenkins      | Jenkins initial setup is required. An admin user has been created and a password generated.
jenkins      | Please use the following password to proceed to installation:
jenkins      |
jenkins      | 45638c79cecd4f43962da2933980197e
jenkins      |
jenkins      | This may also be found at: /var/jenkins_home/secrets/initialAdminPassword
jenkins      |
jenkins      | *************************************************************
```

Pour accéder au mot de passe à partir du conteneur.

```
docker exec -it jenkins sh
/ $ cat /var/jenkins_home/secrets/initialAdminPassword
```

Après avoir entré le mot de passe, nous téléchargerons les plugins recommandés et définirons un ''utilisateur administrateur''.

![](images/005.png)

![](images/006.png)

![](images/007.png)

Après avoir cliqué sur les boutons **Enregistrer et terminer** et **Commencer à utiliser Jenkins**, nous devrions voir la page d'accueil Jenkins. L'un des sept objectifs énumérés ci-dessus est que nous devons avoir la capacité de construire une image dans le Jenkins en cours de dockerisation. Examinez les définitions de volume du service Jenkins dans le fichier de composition.
```
- /var/run/docker.sock:/var/run/docker.sock
```

Le but est de communiquer entre le ''Docker Daemon'' et le ''Docker Client'' (_nous l'installerons sur Jenkins_) sur le socket. Comme le client docker, nous avons également besoin de ''Maven'' pour compiler l'application. Pour l'installation de ces outils, nous devons effectuer les configurations ''Maven'' et ''Docker Client'' sous _Manage menu Jenkins -> Global Tool Configuration_.

![](images/008.png)

Nous avons ajouté les ''installateurs Maven et Docker'' et avons coché la case ''Installer automatiquement''. Ces outils sont installés par Jenkins lors de la première exécution de notre script ![Jenkinsfile](https://github.com/hakdogan/jenkins-pipeline/blob/master/Jenkinsfile). Nous donnons des noms ''myMaven'' et ''myDocker'' aux outils. Nous accéderons à ces outils avec ces noms dans le fichier de script.

Étant donné que nous allons effectuer certaines opérations telles que ''checkout codebase'' et ''push an image to Docker Hub'', nous devons définir les ''Docker Hub Credentials''. Gardez à l'esprit que si nous utilisons un **référentiel privé**, nous devons définir ''Informations d'identification Github''. Ces définitions sont effectuées sous _Jenkins Page d'accueil -> Informations d'identification -> Informations d'identification globales (sans restriction) -> menu Ajouter Credentials_.

![](images/009.png)

We use the value we entered in the ``ID`` field to Docker Login in the script file. Now, we define pipeline under _Jenkins Home Page -> New Item_ menu.

![](images/010.png)

In this step, we select ``GitHub hook trigger for GITScm pooling`` options for automatic run of the pipeline by ``Github hook`` call.

![](images/011.png)

Also in the Pipeline section, we select the ``Pipeline script from SCM`` as Definition, define the GitHub repository and the branch name, and specify the script location (_[Jenkins file](https://github.com/hakdogan/jenkins-pipeline/blob/master/Jenkinsfile)_).

![](images/012.png)

After that, when a push is done to the remote repository or when you manually trigger the pipeline by ``Build Now`` option, the steps described in Jenkins file will be executed.

![](images/013.png)

## Review important points of the Jenkins file

```
stage('Initialize'){
    def dockerHome = tool 'myDocker'
    def mavenHome  = tool 'myMaven'
    env.PATH = "${dockerHome}/bin:${mavenHome}/bin:${env.PATH}"
}
```

The ``Maven`` and ``Docker client`` tools we have defined in Jenkins under _Global Tool Configuration_ menu are added to the ``PATH environment variable`` for using these tools with ``sh command``.

```
stage('Push to Docker Registry'){
    withCredentials([usernamePassword(credentialsId: 'dockerHubAccount', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
        pushToImage(CONTAINER_NAME, CONTAINER_TAG, USERNAME, PASSWORD)
    }
}
```

``withCredentials`` provided by ``Jenkins Credentials Binding Plugin`` and bind credentials to variables. We passed **dockerHubAccount** value with ``credentialsId`` parameter. Remember that, dockerHubAccount value is Docker Hub credentials ID we have defined it under _Jenkins Home Page -> Credentials -> Global credentials (unrestricted) -> Add Credentials_ menu. In this way, we access to the username and password information of the account for login.

## Sonarqube configuration

For ``Sonarqube`` we have made the following definitions in the ``pom.xml`` file of the project.

```xml
<sonar.host.url>http://sonarqube:9000</sonar.host.url>
...
<dependencies>
...
    <dependency>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>sonar-maven-plugin</artifactId>
        <version>2.7.1</version>
        <type>maven-plugin</type>
    </dependency>
...
</dependencies>
```

In the docker compose file, we gave the name of the Sonarqube service which is ``sonarqube``, this is why in the ``pom.xml`` file, the sonar URL was defined as http://sonarqube:9000.
