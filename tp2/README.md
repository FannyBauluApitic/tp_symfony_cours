# TP 2

-   Ajout d'une catégorie
-   Page /category et /category/{id}
-   Création d'un formulaire
-   Traiter le formulaire
-   Validation

---

### Ajout d'une catégorie

Créez une entité **Category** avec les champs suivants :

| Nom         | Type         | Nullable |
| ----------- | ------------ | -------- |
| name        | string (255) | no       |
| description | text         | no       |

Et enfin, ajoutez le champ `articles` avec les propriétés suivantes :

-   Field type : **OneToMany**
-   Related to : **Article**
-   New field name inside Article : **category**
-   Is the Article.category property allowed to be null : **no**
-   Automatically delete orphaned App\Entity\Article objects : **yes**

On demande ici qu'une catégorie puisse contenir plusieurs articles. Regardez l'entité Category qui vient d'être générée ainsi que l'entité Article qui a été modifiée avec le nouveau champ `category`.

Générez une nouvelle migration pour prendre en compte nos derniers changements.

Avant de migrer, nous devons supprimer nos données, en effet, nous demandons un champ `category_id` non null dans notre table `Article`, le problème ici, c'est que nous avons déjà des articles existants. Si nous effectuons notre migration maintenant, Symfony va lever une erreur à ce sujet. Le plus simple est donc de supprimer le schema de votre base de données et de le recréer dans la foulée.

Nous pouvons le faire avec deux commandes fournies par la CLI de Symfony :
```bash
$ php bin/console doctrine:schema:drop --force
$ php bin/console doctrine:schema:create
```

Vous pouvez ensuite effectuer les migrations et vérifier le schéma généré dans votre PHPMyAdmin.

Avant de re-générer nos données, nous avons besoin de créer des catégories. Nous allons le faire via une fixture comme nous l'avons fait pour les articles.

Créez 3 catégories avec Faker dans votre `CategoryFixtures`.

Modifiez ensuite `ArtcleFixtures` avec le code suivant :

```php
public function load(ObjectManager $manager)
{
    $faker = Faker\Factory::create();
    $categories = $manager->getRepository(Category::class)->findAll();

    for ($i = 1; $i <= 10; $i++)
    {
        $article = new Article();

        $sentence = $faker->sentence(4);
        $title = substr($sentence, 0, strlen($sentence) - 1);
        $index = rand(0, count($categories) - 1);
        $category = $categories[$index];

        $article->setTitle($title)
                ->setAuthor($faker->name())
                ->setContent($faker->text(1500))
                ->setCreatedAt($faker->dateTimeThisYear())
                ->setCategory($category);

        $manager->persist($article);
    }

    $manager->flush();
}
```

Si vous chargez vos fixtures comme ceci, un problème devrait apparaître. En effet, Symfony essaye d'exécuter la fixutre `ArticleFixtures` puis `CategoryFixtures`. Seulement, `ArticleFixtures` a besoin que des catégories soient déjà existantes pour créer des articles.

Une solution à ce problème est d'utiliser la méthode `getDependencies` qui doit retourner un tableau de Fixtures à exécuter avant la Fixture dans laquelle nous définissons cette méthode. Pour utiliser cette méthode, votre Fixture doit implémenter l'interface `DependentFixtureInterface`.

```php
public function getDependencies()
{
    return [CategoryFixtures::class];
}
```

---

### Page /category et /category/{id}

Créez un Controller `CategoryController` qui contient :

-   une méthode `index` pour la route **/category** qui liste les différentes catégories
-   une méthode `show` pour la route **/category/{id}** qui affiche tous les articles d'une catégorie

Pour la méthode **index**, vous renverrez la vue `index.html.twig` que vous devez créer dans `templates/category/`.
Sur cette vue sera affichée l'ensemble des catégories. Pour chacune de celles-ci, vous devez afficher :

-   le nom de la catégorie qui contient un lien vers la route **category.show**
-   la description de la catégorie
-   le nombre d'articles contenus dans la catégorie

Pour la méthode **show**, vous renverrez la vue `show.html.twig` que vous devez créer dans `templates/category`.
Sur cette vue sera affichée :

-   le nom de la catégorie
-   le nombre d'articles contenus dans cette catégorie
-   un lien "Retour" vers la page précédente
-   Pour chaque article, vous pouvez reprendre ce que vous avez déjà fait sur la vue `templates/article/index.html.twig`

> Comme toutes les fonctions de votre `CategoryController` auront besoin d'appeler `CategoryRepository`, une bonne pratique consiste à le définir dans le constructeur de la classe plutôt que de le récupérer en paramètre de chaque méthode :

```php
private $repository;

public function __construct(CategoryRepository $repository)
{
    $this->repository = $repository;
}
```

> Vous avez maintenant accès à `$this->repository` dans toutes les méthodes de votre Controller.

Pour la méthode `show`, n'oubliez pas de renvoyer une 404 si aucune catégorie n'est trouvée.

```php
if (!$category) {
    throw $this->createNotFoundException('The category does not exist');
}
```

Ajoutez ensuite le nom de la catégorie avec un lien vers celle-ci dans le détail d'un article (route `article.show`).

Vous pouvez également créer une pagination sur la page `category.show` avec 10 articles par page. Dans ce cas là, la pagination sera un peu différente, vous êtes obligé de sortir une variable `articles`, et ne plus utiliser `category.articles` dans notre vue :
```php
$articles = $paginator->paginate(
    $category->getArticles(),
    $request->query->getInt('page', 1),
    10
);
```

---

### Création d'un formulaire

Créez une méthode `create` dans le Controlleur `ArticleController` sur la route `/article/new`. Cette route retourne la vue `create.html.twig` que vous devez créer dans `templates/article/`.

⚠️ Cette méthode doit être définie avant la méthode `show`, sinon Symfony essaiera de convertir "new" en id (paramètre de la méthode show).

Symfony utilise un Form builder, ce qui nous permet de générer les formulaires directement en PHP. Commencez par créer un formulaire via la commande `php bin/console make:form` avec comme attributs :

-   nom : ArticleType
-   entité associée : Article

La convention pour les formulaires dans Symfony, est qu'il finisse par "Type".

Dans votre méthode **create**, vous pouvez maintenant utiliser la fonction `createForm` qui prend en premier argument `ArticleType` et en deuxième argument un nouvel article :

```php
$article = new Article();

$articleForm = $this->createForm(ArticleType::class, $article);

return $this->render('article/create.html.twig', [
    'articleForm' => $articleForm->createView() 
]);
```

On renvoie ensuite la méthode `createView` qui permet ensuite à Twig de l'utiliser.
Pour l'afficher dans Twig, utilisez simplement la méthode `form` en passant `articleForm` en paramètre.

Si vous essayez maintenant de vous rendre sur la page `/article/new`, vous devriez tomber sur une erreur. En effet, Twig affiche les différentes catégories dans un select. Seulement, Twig ne sait pas quoi afficher ici, il appelle la méthode magique `__toString` sur l'entité Catégorie mais celle-ci n'est pas encore définie.

Vous devez donc définir cette méthode dans votre entité `Category`, cette méthode `__toString` doit renvoyer le champ que nous voulons afficher dans le "select", pour nous ce sera le champ `name`.

Rendez-vous ensuite sur la page `/article/new`, à ce stade, vous devez normalement voir le formulaire généré par Symfony.

On peut voir que Symfony génère un formulaire complet avec des champs adaptés :
- Textarea pour le *content*
- Input de type date pour le *createdAt*
- Select généré automatiquement pour les différentes catégories

En examinant le code HTML généré, on peut constater que Symfony injecte également un **CSRF token**, qui permet de soumettre ce formulaire uniquement via notre page. En savoir plus sur le [CSRF](https://symfony.com/doc/current/security/csrf.html).

Nous pouvons commencer par supprimer le champ **createdAt** dans `ArticleType`. Celui-ci sera généré à la création de l'article.

Ajoutez un bouton "submit" dans le formulaire. Pour ajouter du contenu dans notre formulaire, nous devons modifier son implémentation dans Twig :

```html
    {{ form_start(articleForm) }}
        {{ form_row(articleForm.title) }}

        {{ form_row(articleForm.content) }}

        {{ form_row(articleForm.author) }}

        {{ form_row(articleForm.category) }}

        <button type="submit" class="btn btn-primary btn-block">
            Créer l'article
        </button>
    {{ form_end(articleForm) }}
```

Ici, nous devons utiliser la fonction `form_start` et `form_end` de Twig pour placer quelque chose à l'intérieur de ce formulaire. Nous devons ensuite appeler la fonction `form_row` qui contient le label et l'input. Si vous n'appelez pas `form_row` sur l'ensemble des champs que vous injectez dans le formulaire, Twig les ajoutera automatiquement à la fin du formulaire.

Nous pouvons également personnaliser ce template en définissant nous même les labels. Le code suivant remplace la fonction `form_row` :
```html
<div class="form-group">
    <label for="article_title">Titre de l'article</label>
    {{ form_widget(articleForm.title) }}
</div>
```

Nous pouvons maintenant ajouter des attributs aux inputs/selects dans le fichier `ArticleType` :
```php
->add('title', null, [
    'attr' => [
        'class' => 'form-control',
        'placeholder' => 'Titre de l\'article'
    ]
])
```

La fonction `add` du formBuilder prend en premier paramètre le champ de l'entité, en deuxième le type de l'input (laissez null pour avoir la valeur qu'à généré Symfony par défaut) et enfin en troisième paramètre un tableau d'options.

Sur la vue `article/new.html.twig`, profitez en pour rajouter un lien "annuler" qui redirige sur la route `article.index`.
Sur la vue `article/index.html.twig`, ajoutez un bouton "Créer un article" qui redirige sur la route `article.create`.

---

### Traiter le formulaire

Pour récupérer les informations de notre formulaire et les associer directement à notre nouvel article, nous devons utiliser la méthode `handleRequest` sur notre `articleForm` et qui prend en paramètre la `$request`. Nous pouvons ensuite vérifier l'état de l'article courant lorsque le formulaire est envoyé.

```php
$articleForm->handleRequest($request);

if ($articleForm->isSubmitted())
{
    dd($article);
}
```

Nous pouvons voir que le champ **createdAt** reste `null`, nous allons donc devoir le renseigner nous même lorsque le formulaire est soumis.
```php
if ($articleForm->isSubmitted())
{
    $article->setCreatedAt(new \DateTime);
    ...
}
```
Une fois que votre article est complet, utilisez le Manager de Symfony pour sauvegarder l'article ([Voir documentation](https://symfony.com/doc/current/doctrine.html#persisting-objects-to-the-database)). Vous pouvez maintenant rediriger l'utilisateur sur ce nouvel article avec la fonction `redirectToRoute` qui prend en premier paramètre le nom de la route et en second les paramètres de cette route.

```php
return $this->redirectToRoute('article.show', [
    'id' => $article->getId()
]);
```

---

### Validation

Actuellement, nous pouvons créer un article avec un titre d'une seule lettre, idem pour le contenu et pour l'auteur, voyons comment ajouter une validation côté serveur sur ces différents champs.

Avec Symfony, la validation se fait directement dans les entités. Voir la [documentation](https://symfony.com/doc/current/validation.html).

Dans l'entité Article, commencez par ajouter le Namespace suivant : 

```php
use Symfony\Component\Validator\Constraints as Assert;
```

La convention dans Symfony est de renommer la classe `Constraints` en `Assert` au moment de l'import pour plus de lisibilité dans les Models.

Commençons par utiliser l'assertion [Length](https://symfony.com/doc/current/reference/constraints/Length.html) :

```php
/**
 * @ORM\Column(type="string", length=255)
 * @Assert\Length(min=5, minMessage="Le titre doit faire au minimum {{ limit }} caractères.")
*/
private $title;
```

Il faut maintenant modifier ArticleController pour vérifier que le formulaire est valide :
```php
if ($articleForm->isSubmitted() && $articleForm->isValid())
{
    // ...
}
```

Par défaut, Symfony ajoute une règle de validation côté client et nous empêche donc de vérifier notre validation.  
Vous pouvez supprimer l'attribut `pattern=".{5,}"` et renvoyer le formulaire. La page se recharge, mais rien ne se passe, en effet, nous n'avons pas encore affiché nos erreurs.  
Vous pouvez le faire via la fonction `form_errors()` qui prend en paramètre le champ en question.  

```html
<div class="form-group">
    <label for="article_title">Titre de l'article</label>
    {{ form_widget(articleForm.title) }}
    <small class="text-danger">
        {{ form_errors(articleForm.title) }}
    </small>
</div>
```

Essayez à nouveau, vous devriez voir votre erreur de validation qui apparaît.  
Vous pouvez maintenant ajouter cette règle de validation sur les différents champs du formulaire.

Si vous avez terminé, vous pouvez passer au [TP 3](tp3.md).
