HelloWorld Servlet avec son Dockerfile correspondant

Utilisez d'abord Maven Build pour créer un fichier war dans le dossier Target.

mvn clean package

Artifact sera cree dans le dossier target.

docker build -t mavenbuild .

Une fois cela fait, vous verrez l'image en utilisant docker images

Faites la commande suivante pour lancer le container

docker run -d -p 8080:8080 --name dockercontainer mavenbuild
