# TP 4

- Authentification
- L'entité User
- Formulaire d'inscription
- Page de connexion
- Déconnexion
- Protéger des routes

---

### Authentification

L'authentification est un partie importante d'une application Web. Symfony arrive donc avec un composant très complet sur ce sujet : [le composant Security](https://symfony.com/doc/current/security). 

Toute la configuration de ce composant se trouve dans le fichier `config/packages/security.yaml`.  
Dans la section `security`, nous retrouvons plusieurs sous-sections : 

- Les **Providers** définissent l'endroit où sont récupérés les utilisateurs.  
- Les **Firewalls** définissent les composants qui permettent d'authentifier nos utilisateurs.
- **Access_control** définit les niveaux d'accès à notre application en fonction des rôles d'un utilisateur. 

La première chose à faire pour authentifier des utilisateurs est d'avoir une entité qui les représentent.

---

### L'entité User

Créez une entité **User** avec les champs suivants :

| Nom         | Type         | Nullable |
| ----------- | ------------ | -------- |
| email       | string (255) | no       |
| password    | string (255) | no       |

Créez ensuite une migration et migrez celle-ci pour ajouter notre entité **User** dans la base de données.

---

### Formulaire d'inscription

Créez un formulaire via la commande `make:form` et nommez le RegisterType, ce formulaire sera lié à l'entité **User**.

Dans ce formulaire, nous souhaitons ajouter un champ `confirm_password`. Vous devez donc le rajouter dans ce formulaire mais également dans l'entité User avec son getter et son setter. Ne rajoutez pas d'annotation sur ce champ, car celui-ci n'a rien à voir avec la base de données.

Créez ensuite un Controlleur que vous nommerez `AuthController`, il sera responsable d'afficher les différentes routes liées à l'authentification : **Register**, **Login** et **Logout**.

Dans ce Controller, supprimez la méthode `index` et créez une méthode `register` avec la route `/register` et qui a pour nom `auth.register`.  
Cette méthode utilise `createForm` avec le `RegisterType`, et un nouvel utilisateur.
Vous renverrez ensuite la méthode `createView` sur ce formulaire à votre vue `auth/register.html.twig`.

Sur la vue, affichez les trois champs (email, password, confirm_password) et un bouton "submit". Ajoutez les classes Bootstrap dans le fichier `RegisterType` comme vous l'avez fait pour `ArticleType.`  
Nous devons également préciser que les champs *password* et "confirm_password" sont des inputs de type *password*. En effet, si vous essayez de renseigner ces champs dans votre formulaire, nous pouvons voir que ce sont des inputs de type *text*.

Pour cela, il suffit de modifier la deuxième valeur de la méthode `add` dans notre `RegisterType`:

```php
    ->add('password', PasswordType::class, [
        'attr' => [
            'class' => 'form-control'
        ]
    ])
```
Vous pouvez également ajouter le type `EmailType::class` pour le champ email. 

Nous pouvons maintenant ajouter une validation côté serveur sur nos différents champs :
- Assert\Email sur le champ **email**
- Assert\Length avec au minimum 8 caractères sur le champ **password**
- Assert\EqualTo(propertyPath="password") sur le champ **confirm_password**

Rajoutez un message dans l'assertion EqualTo, sinon Symfony affichera le message par défaut : `This value should be equal to "...".` qui révèle le mot de passe précédemment renseigné.

Vous pouvez également ajouter une validation `Unique` sur le champ email pour que celui-ci soit unique dans notre base de données, cette validation se met directement au niveau de l'entité et n'utilise pas Assert :
```php
use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;

/**
 * @ORM\Entity(repositoryClass="App\Repository\UserRepository")
 * @UniqueEntity(
 *  fields = {"email"},
 *  message = "L'email que vous avez indiqué est déjà utilisé"
 * )
 */
class User
```

---


L'affichage de notre formulaire est maintenant complet, nous passons passer au traitement des données.
Dans le Controller, récupérez la `$request` et le `$manager` via l'injection de dépendances.
Utilisez la méthode `handleRequest` en passant la `$request` en paramètre et vérifiez ensuite si le formulaire est soumis et valide.
Si oui, enregistrez ce nouvel utilisateur dans la base de données.

Vous pouvez voir sur votre PHPMyAdmin que votre utilisateur a son mot de passe stocké en clair. C'est une très grosse faille de sécurité, si quelqu'un récupère votre base de données, il récupère également tous les mots de passe de vos utilisateurs.

Symfony nous fournit une méthode pour "hasher" les mots de passe, via une nouvelle clé à rajouter dans le fichier `security.yaml`: les **Encodeurs**. Vous pouvez définir un encodeur pour notre entité **User** de cette façon :

```yaml
security:
    encoders:
        App\Entity\User:
            algorithm: auto
```

Nous utilisons ici l'algorithme `auto` qui est celui par défaut de Symfony pour hasher une chaîne de caractères.
De retour dans la méthode `register` du `AuthController`, vous devez préciser le champ à encoder. Pour cela, vous devez récupérer l'encodeur depuis l'interface `UserPasswordEncoderInterface` et appeler sa méthode `encodePassword` :

```php
public function register(UserPasswordEncoderInterface $encoder /* .... */)
{
    // ....
    if ($form->isSubmitted() && $form->isValid())
    {
        $hash = $encoder->encodePassword($user, $user->getPassword());
        $user->setPassword($hash);

        // ...
    }
}
```

Si vous essayez de soumettre votre formulaire à nouveau, vous allez avoir une grosse erreur. En effet, l'encodeur a besoin que l'utilisateur qu'on lui donne implémente une interface bien précise : **UserInterface**.

```php
class User implements UserInterface
```

Maintenant que l'entité User implémente **UserInterface**, vous devez définir toutes les méthodes de l'interface dans votre entité. Vous pouvez aller voir directement dans la classe les méthodes qu'elle vous oblige à définir.  
Pour la méthode `getUsername`, retournez `$this->email`.  
Pour la méthode `getRoles`, vous pouvez retourner `['ROLE_USER']`, en effet, nous ne ferons pas de gestion de rôles dans ce cours.   
Enfin pour les méthodes `eraseCredentials` et `getSalt`, ne renvoyez rien.

Vous pouvez ensuite tester l'inscription d'un utilisateur. Vérifiez ensuite dans PHPMyAdmin le mot de passe de l'utilisateur, celui-ci doit être "hashé".

--- 

### Page de connexion

Maintenant que nous avons un utilisateur, il faut maintenant l'identifier sur le site.
Dans votre fichier `AuthController`, créez une méthode `login` avec comme route `/login` et `auth.login` comme nom.   
Cette méthode doit renvoyer la vue `auth/login.html.twig`.

Vous pouvez ajouter un lien vers cette route dans votre fichier `_nav.html.twig`.

Pour connecter des utilisateurs dans Symfony, nous avons besoin de modifier le fichier `security.yaml`.  
Dans la section `firewalls` puis `main`, nous avons besoin de rajouter un `form_login` avec deux attributs :
- login_path : le nom de la route qui affiche le formulaire (GET)
- check_path : le nom de la route qui traite le formulaire (POST)

Dans notre cas, le nom de la route est identique, plus d'informations sur le [form_login de Symfony](https://symfony.com/doc/current/security/form_login.html)

```yaml
form_login:
    login_path: auth.login
    check_path: auth.login
```

Sur la vue `auth/login.html.twig`, nous pouvons maintenant créer un formulaire directement en HTML. Symfony attendra deux champs spécifique, à savoir `_username` et `_password`.

```html
<form action="{{ path('auth.login') }}" method="POST">
    <!-- ... -->
    <input name="_username" ... >
    <!-- ... -->
    <input name="_password" ... >
    <!-- ... -->
</form>
```

Lorsqu'on soumet ce formulaire, rien ne se passe. En fait nous n'avons pas encore précisé à Symfony le [Provider](https://symfony.com/doc/current/security/user_provider.html#entity-user-provider) à utiliser pour récupérer nos utilisateurs.    Dans le fichier `security.yaml`, définissez le Provider `in_database`, avec votre entité User et le champ qui sera utilisé pour l'authentification, pour nous ce sera l'email.  
Vous pouvez ensuite préciser que vous voulez utiliser ce Provider dans la section `main` :

```yaml
providers:
    in_database:
        entity:
            class: App\Entity\User
            property: email

# ...
main:
    # ...
    provider: in_database
```

Réessayez ensuite de vous connecter. Si vous saisissez des identifiants corrects, Symfony vous redirigera automatiquement sur la page d'accueil. Il est possible de modifier la route appelée après l'authentification dans le `form_login` via la propriété `default_target_path` qui prend en paramètre un nom de route.

Vous pouvez voir dans la DebugBar de Symfony si vous êtes connecté et, si oui, avec quel utilisateur.

Actuellement, nous affichons aucune erreur si les identifiants sont invalides. Pour ajouter les erreurs, injectez la dépendance `AuthenticationUtils` dans la méthode `login`. Vous pouvez ensuite appeler la méthode `getLastAuthenticationError` pour récupérer l'erreur et la passer à votre vue.

Dans la vue, nous pouvons l'afficher si celle-ci existe :
```html
{% if error %}
    <div class="alert alert-danger">
        {{ error.messageKey | trans(error.messagedata, 'security') }}
    </div>
{% endif %}
```

Nous précisons ici que ce message doit être traduit à l'aide des messages du composant `Security`. Symfony va donc traduire ce message dans la langue du projet. Pour changer la langue par défaut de votre projet, vous devez modifier la variable `default_locale` dans le fichier `config/package/translation.yaml`.

> Plus d'informations sur le formulaire de connexion sur [la documentation](https://symfony.com/doc/current/security/form_login_setup.html).

---

### Déconnexion

Créez une méthode `logout` dans votre `AuthController` avec la route `/logout` et `auth.logout` comme nom. Cette méthode ne fait aucune action, elle a seulement besoin d'être définie pour que nous puissions y faire référence dans le composant `Security`.

Dans le fichier `security.yaml`, ajoutez la propriété `logout` dans la section `main` :
```yaml
    main:
    # ...
        logout:
            path: auth.logout
            target: home
```

On précise ici la route qu'il faut appeler pour se déconnecter et la route qui sera appelée après cette déconnexion.

Regardez dans la Debugbar de Symfony, si vous êtes connecté, vous avez maintenant la possibilité de vous déconnecter.

Nous allons maintenant vérifier si l'utilisateur est connecté, dans ce cas, on affiche un bouton de déconnexion et sinon on affiche un bouton de connexion. Pour savoir si l'utilisateur est connecté, vous pouvez vérifier la variable `app.user` de Twig.
```html
{% if app.user %}
    <li class="nav-item">
        <a class="nav-link" href="{{ path('auth.logout') }}">
            Déconnexion
        </a>
    </li>
{% else %}
...
```

---

### Protéger des routes

De la même manière, cachez les boutons suivants aux personnes qui ne sont pas connectées :
- bouton **Créer un article** sur la page */article*
- bouton **Modifier un article** sur la page */article/{slug}*
- bouton **Supprimer un article** sur la page */article/{slug}*

Le problème ici, c'est que nous avons uniquement caché les boutons pour accéder aux différentes pages, mais si on rentre directement l'url `/article/new`, la page de création d'un article s'affiche bien, même si nous ne sommes pas connectés.

Pour protéger une route, il faut modifier le fichier `security.yaml`, tout en bas, dans la section `access_control`. Pour protéger notre route `article.new`, nous pouvons ajouter la règle suivante :

```yaml
access_control:
    - { path: /article/new, roles: ROLE_USER }
```

Ici, seul un utilisateur qui possède le rôle `ROLE_USER` pourra accéder à la page `/article/new`. Pour effectuer cette vérification, Symfony va utiliser la méthode `getRoles` sur notre entité `User`.

Si vous essayez d'accéder à la page `/article/new`, Symfony vous redirigera automatiquement sur la page de connexion.  
Pour restreindre l'accès d'une route qui possède un argument dans son url, vous pouvez utiliser `.*` à la place de l'argument.

Vous pouvez en découvrir plus sur la propriété `access_control` sur [la documentation](https://symfony.com/doc/current/security/access_control.html).

> ### Aller plus loin
> Pour ceux qui finissent en avance, essayez de mettre en place une gestion de rôles dans votre application.  
> Essayez par exemple de rajouter un rôle ROLE_ADMIN qui sera le seul à pouvoir supprimer un article. (il faut stocker ce rôle en base de données.)

Si vous avez terminé, vous pouvez passer au [TP 5](tp5.md).