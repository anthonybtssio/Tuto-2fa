- Cloner la repo
- Dans un terminal, faites un : composer install
- Copier/coller le fichier .env dans un fichier .env.local et paramétrer la base de données MariaDB
- Démarrer un serveur avec la commande: symfony server:start -d
- Ajouter un utilisateur via la commande: exemple: symfony console app:add-user nom.prenom@truc.fr motdepasse --role ROLE_ADMIN --role ROLE_ETUDIANT

# 🔐 Authentification à deux facteurs (2FA) avec Symfony & SchebTwoFactorBundle

## ⚙️ Prérequis

* Symfony ≥ 6.4 ou 7.x
* Doctrine ORM pour l’entité `User` (sinon, prévoir un persister personnalisé)
* MariaDB ou autre base de données compatible

---

## 🚀 Installation rapide

1. **Cloner la repo**
2. **Installer les dépendances**

```bash
composer install
```

3. **Configurer la base de données**

* Copier `.env` en `.env.local`
* Modifier la connexion à MariaDB

4. **Démarrer le serveur Symfony**

```bash
symfony server:start -d
```

5. **Ajouter un utilisateur**

```bash
symfony console app:add-user nom.prenom@truc.fr motdepasse --role ROLE_ADMIN --role ROLE_ETUDIANT
```

---

## 📦 Installation du bundle

### 1. Bundle principal

```bash
composer require scheb/2fa-bundle
```

### 2. Packages optionnels

```bash
composer require scheb/2fa-google-authenticator    # Google Authenticator
composer require scheb/2fa-email                   # Code par email
composer require scheb/2fa-backup-code             # Codes de secours
composer require scheb/2fa-trusted-device          # Appareils de confiance
composer require scheb/2fa-totp                    # TOTP standard
```

### 3. Activer le bundle

`config/bundles.php`

```php
return [
    // ...
    Scheb\TwoFactorBundle\SchebTwoFactorBundle::class => ['all' => true],
];
```

### 4. Routes nécessaires

`config/routes/scheb_2fa.yaml`

```yaml
2fa_login:
  path: /2fa
  controller: "scheb_two_factor.form_controller::form"

2fa_login_check:
  path: /2fa_check
```

---

## ⚙️ Configuration

### Fichier `config/packages/scheb_two_factor.yaml`

```yaml
scheb_two_factor:
  trusted_device:
    enabled: false
    lifetime: 5184000
    extend_lifetime: false
    cookie_name: trusted_device
    cookie_secure: false
    cookie_same_site: "lax"
    cookie_domain: ".example.com"
    cookie_path: "/"

  backup_codes:
    enabled: false

  ip_whitelist_provider: null
  two_factor_condition: null
```

### Firewall (`security.yaml`)

```yaml
security:
  firewalls:
    main:
      two_factor:
        auth_form_path: /2fa
        check_path: /2fa_check
        post_only: true
        default_target_path: /
        always_use_default_target_path: false
        auth_code_parameter_name: _auth_code
        trusted_parameter_name: _trusted
        remember_me_sets_trusted: false
        multi_factor: false
        enable_csrf: true
        csrf_parameter: _csrf_token
        csrf_token_id: two_factor
```

---

## 👀 Afficher le code secret dans Twig

Dans ton contrôleur :

```php
return $this->render('security/enable_2fa.html.twig', [
    'form' => $form->createView(),
    'qrCodeContent' => $qrCodeContent,
    'secret' => $user->getGoogleAuthenticatorSecret(),
]);
```

Dans Twig (`enable_2fa.html.twig`) :

```twig
<h2>Code secret pour Google Authenticator</h2>
<p style="font-size: 1.2em; color: #2c3e50;"><strong>{{ secret }}</strong></p>

<p>Ou scannez ce QR Code :</p>
<img src="https://chart.googleapis.com/chart?cht=qr&chs=200x200&chl={{ qrCodeContent }}" alt="QR Code">

{{ form_start(form) }}
{{ form_widget(form) }}
{{ form_end(form) }}
```

✅ Permet à l’utilisateur de saisir manuellement le code ou de scanner le QR Code.

---

## 📝 Étapes globales

1. **Installer le bundle**
2. **Configurer Symfony** (`scheb_two_factor.yaml`)
3. **Adapter l’entité User** (champ secret + interface du bundle)
4. **Définir les routes et firewall**
5. **Gérer l’expérience utilisateur** (QR Code, email/SMS)
6. **Tester le processus**

   * Login classique
   * Login avec 2FA → demande de code OTP
   * Codes invalides / expirés → refusés

Le form = au formulaire 


# 📄 Documentation : `TwoFactorController.php`

## 1️⃣ Namespace et imports

```php
namespace App\Controller;

use App\Entity\User;
use Doctrine\ORM\EntityManagerInterface;
use Scheb\TwoFactorBundle\Security\TwoFactor\Provider\Google\GoogleAuthenticatorInterface;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\Extension\Core\Type\SubmitType;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Annotation\Route;
use Symfony\Component\Security\Http\Attribute\IsGranted;
```

**À quoi ça sert :**

* Déclare le namespace et les classes nécessaires pour :

  * Manipuler l’utilisateur (`User`)
  * Gérer la base de données (`EntityManagerInterface`)
  * Générer et vérifier le code Google Authenticator (`GoogleAuthenticatorInterface`)
  * Créer un formulaire (`TextType`, `SubmitType`)
  * Gérer les requêtes HTTP et les réponses
  * Définir la route et les permissions (`Route`, `IsGranted`)

---

## 2️⃣ Déclaration du contrôleur

```php
final class TwoFactorController extends AbstractController
```

**À quoi ça sert :**

* Classe contrôleur Symfony pour gérer l’activation du 2FA.
* `AbstractController` permet d’utiliser des méthodes pratiques comme `getUser()`, `addFlash()`, `render()`.

---

## 3️⃣ Route et sécurité

```php
#[Route('/2fa/enable', name: 'app_2fa_enable')]
#[IsGranted('ROLE_USER')]
```

**À quoi ça sert :**

* Déclare la route `/2fa/enable` accessible depuis le navigateur.
* `IsGranted('ROLE_USER')` : seuls les utilisateurs connectés peuvent accéder à cette page.

---

## 4️⃣ Récupération de l’utilisateur connecté

```php
$user = $this->getUser();

if (!$user) {
    throw $this->createAccessDeniedException();
}
```

**À quoi ça sert :**

* `$user` récupère l’utilisateur actuellement connecté.
* Si aucun utilisateur connecté, une exception est levée pour empêcher l’accès à cette page.

---

## 5️⃣ Vérification si le 2FA est déjà activé

```php
if ($user->isGoogleAuthenticatorEnabled()) {
    $this->addFlash('info', 'Le 2FA est déjà activé.');
    return $this->redirectToRoute('app_home');
}
```

**À quoi ça sert :**

* Si l’utilisateur a déjà activé le 2FA, il est redirigé vers la page d’accueil avec un message d’information.
* `addFlash()` : permet d’afficher un message temporaire à l’utilisateur.

---

## 6️⃣ Génération du secret Google Authenticator

```php
if (!$user->getGoogleAuthenticatorSecret()) {
    $secret = $googleAuthenticator->generateSecret();
    $user->setGoogleAuthenticatorSecret($secret);
    $em->persist($user);
    $em->flush();
}
```

**À quoi ça sert :**

* Crée un **code secret unique** pour l’utilisateur s’il n’en a pas déjà.
* Ce secret sera utilisé pour générer le QR Code et les codes temporaires dans Google Authenticator.
* Persist et flush : sauvegarde le secret en base de données.

---

## 7️⃣ Génération du QR Code

```php
$qrCodeContent = $googleAuthenticator->getQRContent($user);
```

**À quoi ça sert :**

* Génère le contenu du QR Code à scanner dans Google Authenticator.
* Permet à l’utilisateur de configurer son application facilement.

---

## 8️⃣ Création du formulaire 2FA

```php
$form = $this->createFormBuilder()
    ->add('code', TextType::class, [
        'label' => 'Code (6 chiffres)',
        'required' => true,
    ])
    ->add('submit', SubmitType::class, ['label' => 'Activer'])
    ->getForm();
```

**À quoi ça sert :**

* Formulaire simple pour que l’utilisateur saisisse le **code temporaire généré par Google Authenticator**.
* Validation côté serveur : le code doit correspondre au secret.

---

## 9️⃣ Gestion du formulaire

```php
$form->handleRequest($request);

if ($form->isSubmitted() && $form->isValid()) {
    $code = $form->get('code')->getData();

    if ($googleAuthenticator->checkCode($user, $code)) {
        $user->setGoogleAuthenticatorEnabled(true);
        $em->persist($user);
        $em->flush();

        $this->addFlash('success', '2FA activé avec succès.');
        return $this->redirectToRoute('app_home');
    } else {
        $this->addFlash('danger', 'Code invalide. Réessaye.');
    }
}
```

**À quoi ça sert :**

* Récupère les données saisies par l’utilisateur.
* Vérifie le code avec la méthode `checkCode()`.
* Si le code est correct :

  * Active le 2FA sur l’utilisateur (`setGoogleAuthenticatorEnabled(true)`)
  * Sauvegarde en base de données
  * Affiche un message de succès et redirige
* Sinon, affiche un message d’erreur.

---

## 10️⃣ Affichage de la page Twig

```php
return $this->render('security/enable_2fa.html.twig', [
    'form' => $form->createView(),
    'qrCodeContent' => $qrCodeContent,
    'secret' => $user->getGoogleAuthenticatorSecret(),
]);
```

**À quoi ça sert :**

* Passe les informations au template Twig pour l’affichage :

  * `$form` : formulaire de saisie du code
  * `$qrCodeContent` : QR Code à scanner
  * `$secret` : code secret à afficher au cas où l’utilisateur veuille le saisir manuellement

---

### 🔹 Résumé pour la doc

* Ce contrôleur permet à un **utilisateur connecté** d’activer le 2FA via Google Authenticator.
* Il **génère un secret unique**, **affiche un QR Code**, et **vérifie le code saisi** avant de l’activer.
* Tous les messages (success / info / danger) sont affichés via `addFlash()` pour une bonne UX.
* Les données sensibles sont sécurisées dans la base et ne sont jamais exposées côté serveur.



# 🔐 Réinitialisation de mot de passe (Forgot Password)

Cette fonctionnalité permet aux utilisateurs de réinitialiser leur mot de passe en cas d’oubli, de manière **sécurisée**, en utilisant le **Symfony ResetPasswordBundle**.

---

## 1️⃣ Installation du bundle

Installe le bundle via Composer :

```bash
composer require symfonycasts/reset-password-bundle
````

---

## 2️⃣ Génération automatique avec Symfony Maker

```bash
bin/console make:reset-password
```

Le Maker génère automatiquement :

* L’entité `ResetPasswordRequest` et son repository
* Le contrôleur pour gérer les demandes de réinitialisation
* Les templates Twig pour l’email et le formulaire de réinitialisation
* Le fichier de configuration `config/packages/reset_password.yaml`

---

## 3️⃣ Configuration principale

Dans `config/packages/reset_password.yaml` :

```yaml
symfonycasts_reset_password:
  request_password_repository: App\Repository\ResetPasswordRequestRepository
  lifetime: 3600            # Durée de validité du lien (en secondes)
  throttle_limit: 3600      # Intervalle minimal entre deux demandes
  enable_garbage_collection: true # Supprime automatiquement les demandes expirées
```

---

## 4️⃣ Configuration du mailer en développement

Dans `.env.local` :

```env
MAILER_DSN=smtp://localhost:1025
```

Utilise **MailCatcher** pour intercepter les emails :

```bash
docker run -p 1080:1080 -p 1025:1025 sj26/mailcatcher
```

* **Port 1080** → Interface web pour consulter les emails
* **Port 1025** → SMTP pour envoyer les emails depuis Symfony

---

## 5️⃣ Workflow utilisateur

1. L’utilisateur clique sur **« Mot de passe oublié ? »** sur la page de connexion.
2. Il saisit son **email** dans le formulaire.
3. Un **email de réinitialisation** est envoyé (intercepté par MailCatcher en dev).
4. L’utilisateur clique sur le **lien de réinitialisation** dans l’email.
5. Il est redirigé vers un formulaire pour saisir un **nouveau mot de passe**.
6. Le mot de passe est mis à jour en base et l’utilisateur peut se connecter avec son nouveau mot de passe.

---

## 6️⃣ Vérifications et bonnes pratiques

* Exécuter les migrations Doctrine pour créer la table `reset_password_request` :

```bash
php bin/console make:migration
php bin/console doctrine:migrations:migrate
```

* Assurer que Symfony et PhpStorm utilisent **la même base de données** (`DATABASE_URL` dans `.env.local`).
* Ne jamais révéler si un email existe ou non dans la base de données.
* Vérifier que les anciennes demandes expirées sont bien supprimées (garbage collection).
* Supprimer manuellement les demandes d’un utilisateur si nécessaire :

```php
$repository->removeRequests($user);
```

---

## 7️⃣ Accès au formulaire de réinitialisation

```
http://localhost:8000/reset-password
```

* Accessible via le lien **« Mot de passe oublié ? »** sur la page de connexion.
* En développement, les emails sont interceptés par **MailCatcher** pour visualisation.

---

## 8️⃣ Schéma simplifié du workflow

```
[Page de connexion]
       │
       ▼
[Mot de passe oublié ?] --> [Formulaire email]
       │
       ▼
[Email de réinitialisation] (MailCatcher en dev)
       │
       ▼
[Lien vers formulaire de nouveau mot de passe]
       │
       ▼
[Mot de passe mis à jour en base]
       │
       ▼
[Connexion possible avec le nouveau mot de passe]
```

````

✅ **Explications :**

- Tous les blocs de code utilisent **```bash```**, **```env```**, **```yaml```** ou **```php```** selon le type de contenu.  
- Les titres Markdown sont corrects (`#`, `##`, `###`).  
- Les listes sont correctement formatées.  

Important !! => dans le fichier packages => messager.yaml mettre en commentaire la premiere ligne de code comme cela :
routing:
            # Symfony\Component\Mailer\Messenger\SendEmailMessage: async
            Symfony\Component\Notifier\Message\ChatMessage: async
            Symfony\Component\Notifier\Message\SmsMessage: async





## 🔐 Système de réinitialisation de mot de passe

Le projet utilise le **SymfonyCasts ResetPasswordBundle** afin de gérer un système complet et sécurisé de réinitialisation de mot de passe.  

---

## 📑 Table des matières
- [Création du package reset_password.yaml](#-création-du-package-reset_passwordyaml)
- [Contrôleur ResetPasswordController.php](#-contrôleur-resetpasswordcontrollerphp)
- [Processus utilisateur](#-processus-utilisateur)
- [Enregistrement du nouveau mot de passe](#-enregistrement-du-nouveau-mot-de-passe)
- [Entity ResetPasswordRequest](#-entity-resetpasswordrequest)
- [Envoi du mail](#-envoi-du-mail)
- [Template](#-template)
- [Apports du Bundle](#-apports-du-bundle)
- [Conclusion](#-conclusion)

---

## 📌 Création du package reset_password.yaml
Création d’un package `reset_password.yaml` ⇒ on retrouve la validité du lien pour reset le mot de passe.  

---

## 📌 Contrôleur ResetPasswordController.php
Dans le contrôleur `ResetPasswordController.php` ⇒ on a la création d’une route reset-password. ensuite la classe ResetPasswordController hérité de la classe mère AbstractController.  
Ce controller contient l’envoie du mail avec le lien de réinitialisation. Puis la possibilité de saisir un nouveau mots de passe et de l’enregistrer en base de données.  

---

## 📨 Processus utilisateur
Quand l’utilisateur clique sur “j’ai oublié mon mot de passe” il arrive sur la route reset-password.  
Ensuite un formulaire apparait (`ResetPasswordRequestFormType`) qui lui demande son email. Si l’email est valide, on appelle la méthode `processSendingPasswordResetEmail()` pour envoyer un mail.  
C’est la fonction `request()`.  

Ensuite l’utilisateur est redirigé vers la route `/reset-password/check-email` qui permet de vérifier si l’email entrée est valide. Après avoir entré son email, l’utilisateur est redirigé vers une page de confirmation. Même si l’email n’existe pas dans la base, on affiche toujours la même page.  
C’est la fonction `checkEmail()`.  

Ensuite le lien de réinitialisation est envoyé dans un email. L’utilisateur reçoit un email avec un lien avec un token. Ce token est une clé unique qui prouve que l’utilisateur a bien le droit de changer le mot de passe. La méthode `reset()` vérifie si le token est valide.  

- ✅ Si oui → on affiche un formulaire pour mettre un nouveau mot de passe.  
- ❌ Si non → on redirige vers la demande de réinitialisation.  

C’est la fonction `reset()`.  

---

## 🔑 Enregistrement du nouveau mot de passe
Enregistrement du nouveau mot de passe ⇒ Quand l’utilisateur soumet le formulaire :  
- On supprime le token (il n’est utilisable qu’une seule fois).  
- On hash (chiffre) le nouveau mot de passe.  
- On l’enregistre en base (`$this->entityManager->flush()`).  
- On nettoie la session.  
- Puis on redirige l’utilisateur vers l’accueil.  

---

## 🗄️ Entity ResetPasswordRequest
L’Entity `ResetPasswordRequest` sert à :  
Créer une table en base de données qui enregistre les demandes de réinitialisation. Stocker pour chaque demande :  
- l’utilisateur concerné,  
- le token unique (hashé),  
- la date d’expiration.  

Pouvoir ensuite vérifier si le lien reçu par email est encore valide et appartient bien au bon utilisateur.  
L’entité `ResetPasswordRequest` sert à enregistrer en base de données chaque demande de réinitialisation de mot de passe, avec l’utilisateur concerné, le token unique et la date d’expiration du lien.  

---

## ✉️ Envoi du mail
L’envoi du mail ⇒ Dans `processSendingPasswordResetEmail()` :  
- On vérifie si l’email correspond à un utilisateur.  
- Si oui, on génère un token et on crée un mail avec `TemplatedEmail`.  
- Ce mail contient le lien pour réinitialiser le mot de passe.  
- Enfin, on stocke le token dans la session et on redirige vers la page de confirmation.  

---

## 🎨 Template
Le `reset.html.twig` sert pour l’affichage de la page.  

---

## ⚡ Apports du Bundle
Dans ton Entity `ResetPasswordRequest`, tu vois qu’on utilise :  

```php
class ResetPasswordRequest implements ResetPasswordRequestInterface
{
    use ResetPasswordRequestTrait;
}
````

Ça vient du bundle : au lieu d’écrire toi-même les champs expiresAt, selector, hashedToken + leurs méthodes, le bundle les fournit déjà.

Dans ton contrôleur, tu as :

```php
use SymfonyCasts\Bundle\ResetPassword\Controller\ResetPasswordControllerTrait;
use SymfonyCasts\Bundle\ResetPassword\ResetPasswordHelperInterface;
```

Ça aussi vient du bundle :

* `ResetPasswordControllerTrait` = ajoute des fonctions prêtes (stockage/lecture de token dans la session, nettoyage après reset).
* `ResetPasswordHelperInterface` = génère et valide les tokens automatiquement.

---

## ✅ Conclusion

Le projet utilise le **SymfonyCasts ResetPasswordBundle**, un module externe qui fournit tout ce qui est nécessaire pour gérer un système sécurisé de réinitialisation de mot de passe.
Il évite de tout coder à la main : génération de token, expiration, sécurité, envoi d’email.

```


