# BIG DATA

Florian BONHOMME

22/11/2021

## Notes de cours

- docker run -d -p 9999:80 nginx
- docker ps -> image qui tourne
- docker ps -a -> all 
- docker stop [id]
- docker start [id]
- docker run --name [name] 80:9999 -> nom pour le serv
- docker images -> images pull 
- docker rm [name | id] -> supprimer un conteneur
- docker rmi [name] -> supprimer une image
- docker run [package name]
- docker exec -it -> permet de lancer en ligne de commande
- docker logs [name] -> log du serveur

### Exemple avec mysql :
docker run --name=MySQL -e MYSQL_ROOT_PASSWORD=root -d mysql
docker exec -it MySQL mysql -password

### Exercice (installation JAVA):
docker run -it ubuntu bash
root:/#  apt update
apt install default-jre -> installation JAVA
apt-get install vim

Pour sortir de vim : ":qa"

Pour inspecter un conteneur et avoir des détails sous forme d'un fichier json 
docker inspect [id]


### Exercice modifier index html nginx

docker run -p 9995:80 -d nginx
docker exec -it [id] bash
apt install systemctl
apt install nano
cd /usr/share/nginx/html/
nano index.html
on modifie un truc de la page
systemctl restart nginx

### Mappage volume
docker run -dti -p 8080:80 -v C:\Users\floma\Documents\docker\backupnginx:/usr/share/nginx/html/ nginx
Sur chrome : error 403
Explorateur de fichier : C:\Users\floma\Documents\docker\backupnginx
On crée un fichier index.thml et on refresh la page 

## Création d'une image avec nodejs

Dispo ici : https://hub.docker.com/r/florianbonhomme/node-bonhomme

1. On crée un Dockerfile

```dockerfile
FROM node:12-alpine
RUN apk update
RUN apk add nodejs
COPY . .
RUN npm install
CMD ["node","index.js"]
```

2. Création d'image

    docker build -t node-bonhomme .

3. Lancer l'image

    docker run -d -p 3000:3000 node-bonhomme

4. Déposer l'image sur dockerhub

    docker build -t florianbonhomme/node-bonhomme .
    docker push florianbonhomme/node-bonhomme

5. Pour run depuis un autre poste

    docker run -d  -p 3000:3000 florianbonhomme/node-bonhomme
    

# Creation de d'un docker-compose pour un projet Springboot et MongoDB

Florian BONHOMME

29/11/2021

Dispo ici : https://hub.docker.com/r/florianbonhomme/bigdata-bonhomme

## Création du projet 
Dans le cadre du déploiement automatisé on à crée un projet SpringBoot avec les dépendances Web et MongoDB pour pouvoir créer un API Rest basique qui renvoit des produits stockés dans une base de donnée MongoDB.

1. On définie une entitée Produit avec des champs basiques de cette façon :
```java
@Document(collection = "produit")
public class Produit implements Serializable {
    @Id
    private String id;

    @Field("designation")
    private String designation;

    @Field("prix")
    private double prix;

    public Produit(String id, String designation, double prix) {
        this.id = id;
        this.designation = designation;
        this.prix = prix;
    }

    public Produit() {
    }

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getDesignation() {
        return designation;
    }

    public void setDesignation(String designation) {
        this.designation = designation;
    }

    public double getPrix() {
        return prix;
    }

    public void setPrix(double prix) {
        this.prix = prix;
    }
}
```

Où on map les champs en fonction de Mongo.

2. Repository pour requêter la base de donnée : 

```java 
@Repository
public interface ProduitRepository extends MongoRepository<Produit, String> {
    Produit findByDesignation(String designation);
}

```

3. Controller rest :

```java
@RestController
@RequestMapping("/produit")
public class ProduitService  {

    private final ProduitRepository produitRepository;

    public ProduitService(ProduitRepository produitRepository) {
        this.produitRepository = produitRepository;
    }

    @GetMapping("/{desingation}/{prix}")
    public Produit add(@PathVariable String desingation, @PathVariable double prix)
    {
        if(!desingation.isEmpty())
        {
           return produitRepository.insert(new Produit(null, desingation, prix));
        }
        else
        {
            return null;
        }

    }

    @GetMapping("/{designation}")
    public Produit getById(@PathVariable String designation )
    {
        return produitRepository.findByDesignation(designation);
    }

}
```

## Création de l'image SpringBoot et push sur Docker Hub
Comme dans la partie précédente on à build et push l'image suivant ce Dockerfile

```dockerfile
FROM openjdk:14
ADD bigdata-mongo.jar bigdata-mongo.jar
CMD ["java", "-jar", "bigdata-mongo.jar"]
expose 8080
```

Et on suit le même raisonement que dans le 1er chapitre

## Création du docker-compose.yml pour lier l'application Springboot à MongoDB 

On va utiliser docker-compose pour lier la base de donnée à l'api rest, ici le fichier docker-compose.yml : 
```yml
version: "3.8"
services:
  api-database:
    image: mongo
    container_name: "api-database"
    ports:
      - 27017:27017
  mongo-api:
    image: florianbonhomme/bigdata-bonhomme
    ports:
      - 9091:8080
    links:
      - api-database
```

Ici on effectue les étapes suivantes :
- on choisi la version de docker-compose
- on déclare un service *api-database*
- ce service prendra l'image mongo et s'appelera api-database
- les ports seront mappés sur 27017:27017
- on déclare un service *mongo-api*
- ce service prendra notre image crée précédement qui est sur Docker Hub   
- les ports seront mappés sur 9091:8080
- On lie enfin la base de donnée et l'application
