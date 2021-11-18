# TP 1

-   Notre première page
-   Introduction à Twig
-   Créer des données
-   Utilisons ces données avec Doctrine
-   Affichage d'une page article
-   Mise en place d'une pagination

---

### Notre première page

Créez un Controller `ArticleController` à l'aide de la commande `php bin/console make:controller`.

Cette commande crée également :

-   une route définie dans le Controller (`src/Controller/ArticleController`)
-   une vue dans le dossier `templates/article/`.

On constate que la route est définie sous forme d'annotations directement au-dessus de la méthode. Il existe plusieurs méthodes pour créer des routes avec Symfony (yaml, xml, annotations...). Nous resterons sur le système d'annotations dans ce cours.

Modifiez cette route pour changer le nom par `article.index`.

> **Comment Symfony sait-il où chercher les routes ?**  
> C'est dans le fichier `config/routes/annotations.yaml` qu'on demande à Symfony de rechercher toutes les routes sous formes d'annotations dans le répertoire `src/Controller`.

Disséquons maintenant le code de la méthode `index` du Controller:

```php
public function index()
{
    return $this->render('article/index.html.twig', [
        'controller_name' => 'ArticleController',
    ]);
}
```

Ici, la méthode `render` qui nous permet de renvoyer une vue. Par défaut, Symfony va chercher les vues dans le dossier `templates` (voir la configuration dans `config/packages/twig.yaml`).
Nous chargeons donc ici la vue par défaut, à savoir celle qui a été créée avec ce Controller.  
Le deuxième paramètre de la fonction `render` est un tableau de variables qu'on passe directement à la vue.

Vous l'avez sûrement remarqué, nous utilisons `$this` pour récupérer la méthode `render`. En effet, notre Controller étend la classe `AbstractController` qui utilise le Trait `ControllerTrait`. Vous pouvez regarder dans ces différents fichiers les méthodes misent à disposition dans notre Controller.

Rendez-vous sur `***.lpweb-lannion.fr:7777/article` ou `localhost:7777/article` pour découvrir votre première page créée avec Symfony.

---

### Introduction à Twig

Twig est un moteur de template qui permet d'écrire nos vues très simplement avec une syntaxe plus légère et moins verbeuse que du PHP. Voir la [documentation de Twig](https://twig.symfony.com/).

Jetons un oeil à notre vue dans `templates/article/index.html.twig`.   
La première ligne permet d'étendre d'un layout de base.
Ce layout nous permet de définir des sections globales au site pour ne pas à avoir à les dupliquer (head, menu, footer...).

Il suffit de définir des `block` via `{% block body %}{% endblock %}` dans le layout et je pourrai ensuite les utiliser dans ma vue de la même manière.

Tout le code que j'inclurai dans un block sera intégré au block du layout.

Nous constatons que dans la vue, on affiche `{{ controller_name }}`, c'est une variable qui vient directement du Controlleur, vous pouvez changer sa valeur dans notre Controller et rechargez la page.

Twig utilise la syntaxe `{{ }}` pour afficher quelque chose et `{% %}` pour utiliser une fonction du langage.

Exemple:

```html
<main>
    {% if users | length > 0 %}
        <section class="user">
            {% for user in users %}
                <div class="card-user">
                    <h3>{{ user.name | upper }}</h3>
                    <p>{{ user.description }}</p>
                </div>
            {% endfor %}
        </section>
    {% else %}
        <p>No users have been found...</p>
    {% endif %}
</main>
```

Vous pouvez inclure Bootstrap dans le fichier `base.html.twig` via un CDN.

Ajoutez un fichier `templates/partials/_nav.html.twig` et ajoutez un menu avec trois liens :
- **Accueil** sur "/"
- **Article** sur "/article"
- **Catégorie** sur "/category"

Importez ce fichier `_nav.html.twig` dans le fichier `base.html.twig` avec la fonction `include` de Twig :
```php
{% include "partials/_nav.html.twig" %}
```

---

### Créer des données

Jusqu'ici, nous avons parlé du Controlleur et de la Vue, il manque donc la partie Model (la liaison entre notre application et la base de données).  
Dans Symfony, une entité représente une table. Nous allons commencer par créer une entité **Article**, qui possédera les champs suivants :

| Nom       | Type         | Nullable |
| --------- | ------------ | -------- |
| title     | string (255) | no       |
| author    | string (255) | no       |
| content   | text         | no       |
| createdAt | datetime     | no       |

Pour créer une entité, utilisez la console en vous plaçant dans le conteneur `symfony-php` :
```bash
$ docker exec -it symfony-php bash
```
Puis de lancer :
```bash
$ php bin/console make:entity
```

Deux nouveaux fichiers sont ensuite créés :

-   src/Entity/Article
-   src/Repository/ArticleRepository

Le premier représente la table sous forme d'une classe avec ses getters/setters, on remarque d'ailleurs que Symfony a généré tout le code pour nous.  
Le deuxième fichier représente le "repository", c'est-à-dire le fichier de sélection, c'est la qu'on écrira nos requêtes pour attaquer la base de données.

Effectuez la commande `make:migration` pour demander à Symfony de vérifier s'il existe des différences entre notre structure SQL actuelle et nos fichiers dans le répertoire `src/Entity`. Symfony va ensuite générer une migration contenant des instructions SQL si celui-ci détecte des différences.

Ce fichier de migration se trouve dans le répertoire `src/Migrations`, la fonction `up` ajoute les nouveautés depuis la dernière migration alors que la fonction `down` annule cette migration.
Il faut maintenant lancer la commande `doctrine:migrations:migrate` pour appliquer la fonction `up` de la migration.

Il ne reste plus qu'à vérifier dans PhpMyAdmin que la table Article a bien été ajoutée.

> Ce système de migrations est très performant et utilisé par l'ensemble des Frameworks modernes. Celui-ci permet de récupérer le projet via un repo git, et de lancer la commande `doctrine:migrations:migrate` pour effectuer toutes les migrations du projet et avoir la structure de la base de données à jour avec le projet.

Nous allons maintenant utiliser une librairie qui nous permet de remplir notre table avec une commande Symfony plutôt que d'avoir à créer les articles directement dans PhpMyAdmin. Pour cela nous avons besoin de [orm-fixtures](https://packagist.org/packages/doctrine/doctrine-fixtures-bundle). Nous allons également utiliser une librairie pour générer des données "fake" à notre place, il s'agit de [Faker](https://packagist.org/packages/fzaninotto/faker).

Pour cela, il suffit de se placer dans le conteneur `symfony-php` en lançant :
```bash
$ docker exec -it symfony-php bash
```
Puis de lancer :
```bash
$ composer require orm-fixtures fzaninotto/faker --dev
```

On ajoute ici le `--dev` car ces dépendances ne seront pas utilisées en production.

Nous pouvons maintenant créer notre Fixture `ArticleFixtures` à l'aide de la commande `make:fixtures`. Voilà à quoi ressemble mon fichier :

```php
public function load(ObjectManager $manager)
{
    $faker = Factory::create();

    for ($i = 1; $i <= 10; $i++)
    {
        $article = new Article();

        $sentence = $faker->sentence(4);
        $title = substr($sentence, 0, strlen($sentence) - 1);
        $article->setTitle($title)
                ->setAuthor($faker->name())
                ->setContent($faker->text(3000))
                ->setCreatedAt($faker->dateTimeThisYear());

        $manager->persist($article);
    }

    $manager->flush();
}
```

⚠️ N'oubliez pas d'inclure :
```php
use Faker;
use App\Entity\Article;
```

Je commence par récupérer l'objet Factory de Faker. Je crée ensuite 10 articles dans une simple boucle `for`, à l'intérieur de celle-ci je crée un nouvel Article, j'appelle ensuite ses différents "setters" qui sont définis dans sa classe.
J'utilise ensuite les méthodes de Faker pour me générer des données pertinentes. Voir la [documentation de Faker](https://github.com/fzaninotto/Faker).

Le manager est fourni par l'ORM **doctrine** et celui-ci me permet de persister un article à chaque tour de ma boucle. Je finis par utiliser la fonction `flush`, pour sauvegarder en base de données tout ce qui était stocké dans mon manager (via la commande **persist**).

Pour exécuter cette Fixture, entrez la commande suivante :

```
php bin/console doctrine:fixtures:load
```

Ouvrez ensuite votre PhpMyAdmin pour découvrir les données que nous venons de générer.

---

### Utilisons ces données avec Doctrine

Pour utiliser les données d'une entité, nous avons besoin de son Repository.

```php
$articleRepository = $this->getDoctrine()->getRepository(Article::class);
```

Nous avons maintenant accès au Repository de la classe Article, pour rappel, celui-ci se trouve dans `src/Repository/ArticleRepository`. Par défaut, aucune méthode n'est disponible dans ce fichier. Si on regarde plus en profondeur dans le code, nous pouvons voir que notre Repository étends de la classe `EntityRepository` dans le dossier `/vendor/doctrine/orm/lib/Doctrine/ORM/`.  
Dans ce fichier, nous pouvons voir que beaucoup de méthodes sont définies pour nous : **find**, **findOneBy**, **findAll**...

Commençons par récupérer tous les articles :

```php
$articles = $articleRepository->findAll();
```

Pour constater les données que nous récupérons dans `$articles`, nous pouvons utiliser la fonction `dd` de Symfony, qui permet de faire un `dump`, une version améliorée du `var_dump` de PHP suivi d'un `die`.

```php
public function index()
{
    ...
    $articles = $articleRepository->findAll();
    dd($articles);
    ...
}
```


Symfony nous aide encore une fois et nous propose une autre syntaxe plus simple qui consiste à bénéficier de l'injection de dépendances en récupérant le Repository directement en paramètre de la méthode. En effet, en précisant le type qui est attendu dans la variable, Symfony sait ce qu'il doit nous retourner :

```php
public function index(ArticleRepository $articleRepository)
{
    $articles = $articleRepository->findAll();
    ...
```

Nous pouvons ensuite renvoyer la variable `$articles` à notre vue, comme l'exemple `controller_name`.
Utilisez Twig pour afficher ensuite la liste des articles dans votre vue.

Pour chaque article :
-   affichez son titre avec un lien vers `/article/{id de l'article}`
-   affichez les 300 premiers caractères du contenu suivi d'un `... voir plus` (filtre slice)
-   le **...voir plus** est également un lien vers `/article/{id de l'article}`
-   affichez la date avec un format 24/12/2019 ainsi que le nom de l'auteur

---

### Affichage d'une page article

Commencez par créer une nouvelle méthode `show` dans votre `ArticleController` qui prend en paramètre l'id de l'article avec la route suivante : `@Route("/article/{id}", name="article.show")`

Récupérez l'article associé à l'id reçu via la fonction `find` du Repository. Cette fonction prend par défaut l'id de l'article. Retournez ensuite l'article à la vue `templates/article/show.html.twig`.

Si aucun article n'est trouvé avec l'id passé en paramètre, renvoyez une page 404 :
```php
if (!$article)
{
    throw $this->createNotFoundException('The article does not exist');
}
```

Cette nouvelle vue doit étendre du template `base.html.twig`, vous afficherez : le titre de l'article, son contenu, son auteur et sa date de création avec le format 24/12/2019. Ajoutez également un lieu **retour** qui permet de retourner à la liste des articles.

Profitez-en pour modifier les liens dans notre template `index.html.twig` avec la fonction `path` de Twig plutôt que d'avoir des liens en dur. 
`<a href="{{ path('article.show', {id : article.id}) }}">`
Cette méthode permet de changer les urls des routes sans avoir à modifier toutes les urls dans notre code.

---

### Mise en place d'une pagination

Actuellement, nous n'avons que 10 articles dans notre base de données, modifiez la boucle du fichier `ArticleFixture` pour générer une centaine d'articles et relancez la commande `doctrine:fixtures:load`.

Sur la page `article.index`, nous voyons maintenant que tous nos articles sont chargés, ce n'est pas très pratique. Pour remédier à ce problème, nous allons mettre en place un système de pagination qui affichera 10 articles par page.

Nous allons utiliser le paquet [KnpPaginatorBundle](https://github.com/KnpLabs/KnpPaginatorBundle) pour nous aider à mettre en place cette pagination. 

Utilisez l'image Docker de Composer pour l'installer :
```bash
$ docker run --rm --interactive --tty \
  --volume $PWD:/app \
  --user $(id -u):$(id -g) \
  composer require knplabs/knp-paginator-bundle
```

Dans la méthode `index` de votre Controller, vous pouvez maintenant injecter ce "paginator" :
```php
public function index(ArticleRepository $articleRepository, PaginatorInterface $paginator) 
```

Vous avez accès à une méthode `paginate` sur ce "paginator" qui prend 3 arguments : l'ensemble des articles, la page souhaitée, et le nombre d'articles à afficher par page.
Vous pouvez récupérer la page souhaitée via l'objet `Request` : `$request->query->getInt('page', 1)`. Ici, si aucune page n'est renseignée dans l'url, c'est la première page qui est prise par défaut.

Pour afficher la liste des pages disponibles sur la vue, vous pouvez utiliser la fonction `knp_pagination_render` :
```html
{{ knp_pagination_render(articles) }}
```

Vous pouvez modifier le thème de base par celui de Bootstrap. Pour cela, vous devez ajouter le fichier de configuration `knp_paginator.yaml` dans le dossier `config/packages/`, vous trouverez le contenu du fichier sur le Github du projet.  
Dans la section `template`, modifiez la valeur de `pagination` avec `@KnpPaginator/Pagination/twitter_bootstrap_v4_pagination.html.twig`.

Vous avez maintenant une belle pagination sur votre page. Pour modifier les labels "label_previous" et "label_next", vous devez ajouter un fichier `KnpPaginatorBundle.fr.yaml` dans le dossier `translations` et renseigner une valeur pour ces deux clés :

```yaml
label_next: Suivant
label_previous: Précédent
```

Si vous ne voyez pas de changements, c'est que vous n'avez pas dit à Symfony que votre projet était en Français. Vous pouvez modifier la valeur par défaut dans le fichier `config/packages/translations.yaml`.

Si vous avez terminé, vous pouvez passer au [TP 2](tp2.md).
