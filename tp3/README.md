# TP 3

-   Éditer un article
-   Supprimer un article
-   Alertes flash
-   Compteur de likes
-   Retourner une réponse en JSON
-   Page d'accueil

---

### Éditer un article

Pour continuer sur le CRUD de notre entité Article, nous allons maintenant voir la méthode "update".  
Créez cette méthode **update** dans votre `ArticleController` avec la route `/article/{id}/edit` et `article.edit` comme nom.

Cette route récupère l'article courant avec son id et renvoie la vue `article/update.html.twig`.
Sur la page "article.show", ajoutez un lien "Éditer l'article" qui ramène sur la route `article.update`.

Inspirez vous de la méthode `create` déjà créée, pour créer un formulaire. L'article passé en deuxième paramètre de la fonction `createForm` est ici l'article que nous venons de récupérer et non un nouvel article.  
Une fois sur la vue, nous pouvons constater que les champs sont déjà remplis avec les valeurs de l'article courant.

À la soumission du formulaire, vous ne devez pas mettre à jour la date de création de l'article.

### Supprimer un article

Créez une fonction **delete** dans `ArticleController` avec la route `@Route("/article/{id}/delete", name="article.delete")`.

> Pour afficher les routes de l'application, utilisez la commande `php bin/console debug:router`

Sur la page `article.show`, ajoutez un formulaire avec une action qui pointe vers la route `article.delete`. 
Actuellement, les seules méthodes HTTP supportées par les formulaires HTML sont GET et POST. Pour contrer ce problème, Symfony utilise un input de type `hidden`, avec un *name* qui vaut `_method` et la valeur égale à la méthode HTTP voulue.

Dans notre exemple, cela donnerait :
```html
<input type="hidden" name="_method" value="DELETE">
```
Pour que Symfony interprète ce champ, vous devez préciser un `method="POST"` sur votre formulaire.

Dans le Controller, récupérez ensuite l'article avec son id. Si aucun article n'est trouvé, renvoyez une erreur 404.
Vous pouvez ensuite appeler la méthode `remove` du Manager qui prend en paramètre l'article. Il n'y a plus qu'à appeler la méthode `flush` du Manager pour sauvegarder ses informations dans la base de données.

Rediriger ensuite sur la route `article`.

Nous pouvons ajouter ici la protection CSRF qui empêche d'appeler notre route `article.delete` sans ce fameux jeton, accessible uniquement depuis notre page.
Celui-ci est automatiquement ajouté pour les formulaires générés avec le Form Builder de Symfony.

Dans notre cas, pour le rajouter, nous devons utiliser un input de type hidden, comme pour la méthode DELETE :
```html
<input type="hidden" name="token" value="{{ csrf_token('delete-item') }}">
```
Nous passons ici la chaîne de notre choix dans la méthode `csrf_token` de Twig.

Dans le Controller, nous devons maintenant vérifier si ce CSRF token est valide :

```php
$csrfToken = $request->request->get('token');

if ($this->isCsrfTokenValid('delete-item', $csrfToken))
{
    // ...
}
```

Si ce token n'est pas valide, vous pouvez lever une exception de type `InvalidCsrfTokenException` (n'oubliez pas d'importer le namespace).

Vous pouvez également rajouter un **confirm** directement sur votre formulaire.
```javascript
onsubmit="return confirm('Etes-vous sûr de supprimer cet article ?')"
```

---

### Alertes flash

Nous pouvons maintenant ajouter des alertes flash pour améliorer l'expérience utilisateur lorsque celui-ci vient de faire une action de type `create/update/delete`.  
Les messages flash utilisent la session de l'utilisateur. Ils sont automatiquement supprimés après leur premier affichage, [plus d'informations sur la documentation](https://symfony.com/doc/current/controller.html#flash-messages).

Pour cela, nous pouvons utiliser la méthode `addFlash` dans nos Controlleurs. Cette méthode prend en premier paramètre le niveau de l'alerte et en deuxième le message. Par exemple dans notre méthode `delete` :

```php
    // ...
    $manager->remove($article);
    $manager->flush();

    $this->addFlash('success', "L'article {$article->getTitle()} a bien été supprimé !");

    return $this->redirectToRoute('article.index');
```

Pour afficher ces messages flash, ajoutez ce code dans le fichier `templates/partials/_flash.html.twig` :
```html
{% for label, messages in app.flashes %}
    {% for message in messages %}
        <div class="alert alert-{{ label }}">
            {{ message }}
        </div>
    {% endfor %}
{% endfor %}
```
Ici, nous récupérons le niveau sous la clé **label**, ce qui nous permet d'utiliser les classes Bootstrap en fonction de l'alerte (success, warning, danger...).

Il suffit maintenant d'inclure ce partial dans notre fichier `base.html.twig`, au dessus du block `body`.

```php
{% include "partials/_flash.html.twig" %}
```

Vous pouvez maintenant ajouter un message flash sur la création ainsi que sur la modification d'un article.

---

### Compteur de likes

Dans l'entité **Article**, créez une propriété `likes` avec l'annotation `@ORM\Column(type="integer")`. Cette propriété devra avoir **0** comme valeur par défaut.
Vous devez également créer son getter `getLikes` et deux méthodes pour modifier sa valeur :

-   incrementLikes
-   decrementLikes

Ces deux méthodes ne prennent pas d'arguments et ne font qu'ajouter/enlever un like à la valeur courante de `likes`.

Une fois que cela est fait, effectuez une nouvelle migration et migrez la pour appliquer nos changements en base de données.

Vérifiez ensuite dans PHPMyAdmin que ce nouveau champ est bien présent avec une valeur initiale à 0.

Dans la vue qui affiche un article, ajoutez son nombre de likes ainsi que deux boutons :

```html
<div class="mt-3">
    <button id="decrement" class="btn btn-outline-dark btn-lg">👎</button>
    <strong class="p-5">
        <span id="likes">{{ article.likes }}</span>
        {{ article.likes <= 1  ? 'like' : 'likes' }}
    </strong>
    <button id="increment" class="btn btn-outline-dark btn-lg">👍</button>
</div>
```

Modifiez également le `h1` de la vue en ajoutant un attribut `data-id` comme ceci :

```html
<h1 data-id="{{ article.id }}">{{ article.title }}</h1>
```

En effet, cette technique nous permet ensuite de récupérer l'id de l'article en JavaScript, qui nous sera utile pour la suite.

---

### Retourner une réponse en JSON

Commencez par créer une méthode `like` dans `ArticleController`, avec la route `/article/{id}/like`.

Cette route récupère un article via son id passé en paramètre et si aucun article n'est trouvé, renvoyez une erreur 404. Vous n'utiliserez pas la méthode `createNotFoundException` qui retourne une vue, mais une réponse en JSON comme ceci :

```php
if (!$article)
{
    // throw $this->createNotFoundException('The article does not exist');
    return $this->json('The article does not exist', 404);
}
```

S'il y a un article, vous n'avez qu'à appeler sa méthode `incrementLikes`.
Il faut ensuite persister les modifications via le Manager et appeler la méthode `flush` pour appliquer les changements en base de données.

Vous renverrez ensuite une réponse en JSON plutôt qu'une vue avec comme contenu `likes` qui contient le nombre courant de likes et un code HTTP 200.

```php
return $this->json(['likes' => $article->getLikes()], 200);
```

Vous pouvez ensuite rajouter ce bout de code JavaScript qui permet de faire une requête AJAX au clic sur le bouton **increment** :

> Placez ce code dans le block {% javascripts %} et non dans le block {% body %} (voir le fichier base.html.twig)

```javascript
const id = document.querySelector("h1").getAttribute("data-id");
const likes = document.querySelector("#likes");
const increment = document.querySelector("#increment");
const url = `/article/${id}/like`;

increment.addEventListener("click", () => {
    increment.disabled = true;

    fetch(url)
        .then(res => res.json())
        .then(res => {
            likes.textContent = res.likes;
            increment.disabled = false;
        });
});
```

En cliquant sur le bouton 👍, le compteur doit incrémenter. Rafraîchissez la page pour vérifier que le nombre de likes est bien sauvegardé en base de données.

Il ne vous reste plus qu'à faire la même chose avec le bouton 👎, avec une méthode `unlike` sur le Controller et la route `/article/{id}/unlike`.

Vous pouvez également bloquer le nombre de likes à 0 pour ne pas descendre en négatif.

---

### Page d'accueil

Créez un nouveau Controller `HomeController` avec une méthode `index` qui prendra comme route le "/" du site.
Vous renverrez la vue `home/index.html.twig`.

Sur cette vue, affichez deux colonnes :
- Les 5 articles avec le plus grand nombre de likes
- Les 5 articles les plus récents

Pour chaque élément de la liste, vous afficherez son titre avec un lien sur l'article, son nombre de likes et sa date de création.

Voici à quoi doit ressembler votre méthode `index`:

```php
public function index(ArticleRepository $articleRepository)
{
    return $this->render('home/index.html.twig', [
        'mostLikedArticles' => $articleRepository->findMostLiked(5),
        'mostRecentArticles' => $articleRepository->findMostRecent(5)
    ]);
}
```

Vous devez donc définir ces deux méthodes dans votre `ArticleRepository`. Chaque méthode prend un argument `$number` qui correspond au nombre d'articles que la méthode doit retourner.

Inspirez vous des méthodes disponibles dans le `repository` et de la [documentation](https://symfony.com/doc/current/doctrine.html#querying-with-the-query-builder) pour créer vos propres "Query Builder".

Si vous avez terminé, vous pouvez passer au [TP 4](tp4.md).
