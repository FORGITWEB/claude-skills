---
name: full-audit
description: "Audit fonctionnel end-to-end d'un projet web. Trace chaque flux utilisateur du bouton jusqu'a la BDD/API : UI -> handler -> API route -> service externe/BDD -> retour UI. Detecte les chaînons manquants, les features non connectees, et corrige tout. Usage : /full-audit [chemin-du-projet]"
compatibility: claude-code-only
---

# Full Audit — Audit fonctionnel end-to-end & Fix

Verifie que **chaque action utilisateur** dans l'app est connectee de bout en bout : du clic sur le bouton jusqu'a l'ecriture en BDD/envoi d'email/appel API, et retour au feedback utilisateur. Detecte tous les chainons manquants et les corrige.

## Principe fondamental

**L'audit ne regarde pas si le code est "propre". Il verifie si l'app MARCHE.**

Pour chaque feature de l'app, tracer le flux complet :
```
UI (bouton/formulaire/lien)
  -> Handler JS (onClick/onSubmit)
    -> Appel API (fetch/axios)
      -> API Route (validation + logique)
        -> Service externe (BDD/email/paiement/etc.)
      <- Reponse API
    <- Traitement reponse
  <- Feedback utilisateur (succes/erreur/loading)
```

Si UN SEUL maillon de cette chaine est manquant ou casse = **BUG CRITIQUE**.

## Usage

| Commande | Action |
|----------|--------|
| `/full-audit [chemin]` | Audit complet + corrections |
| `/full-audit [chemin] --report-only` | Rapport uniquement, pas de corrections |

---

## Phase 1 : Cartographie du projet (Agent principal)

Avant tout audit, construire la **carte complete** de l'app.

### 1.1 — Inventaire des pages & routes

```
Pour chaque page dans app/ ou pages/ :
  - URL de la route
  - Composant principal
  - Est-ce une page statique (SSG) ou dynamique (SSR/client) ?
  - Quels composants enfants elle utilise ?
```

### 1.2 — Inventaire des API routes

```
Pour chaque fichier dans app/api/ ou pages/api/ :
  - Route (ex: /api/contact, /api/reservations)
  - Methodes (GET/POST/PUT/DELETE)
  - Ce qu'elle fait (envoyer email, ecrire en BDD, appeler API externe)
  - Services externes utilises (Brevo, Stripe, Supabase, Prisma, etc.)
```

### 1.3 — Inventaire des actions utilisateur

```
Pour chaque page, lister TOUTES les actions possibles :
  - Boutons (CTA, submit, navigation, toggle, etc.)
  - Formulaires (contact, reservation, login, recherche, etc.)
  - Liens (navigation, ancres, externes)
  - Interactions (calendrier, slider, modal, dropdown, etc.)
  - Events (scroll, hover avec effet fonctionnel, etc.)
```

### 1.4 — Inventaire des connexions externes

```
Lister tous les services/APIs/BDD utilises :
  - Base de donnees (Supabase, Prisma, MongoDB, etc.)
  - Email (Brevo, Resend, SendGrid, Nodemailer)
  - Paiement (Stripe, PayPal)
  - Auth (NextAuth, Clerk, Supabase Auth)
  - Stockage (S3, Cloudinary, Vercel Blob)
  - Autres APIs (Google Maps, calendrier, etc.)

Pour chacun :
  - Variable d'env requise
  - Variable d'env presente dans .env.local ? Dans le code Vercel ?
  - Client/SDK initialise correctement ?
```

---

## Phase 2 : Audit des flux end-to-end (Sub-Agents paralleles)

Lancer **3 agents en parallele** avec `Task(subagent_type: "general-purpose")`.

### Agent 1 : Tracage des flux UI -> API

**Le coeur de l'audit.** Pour CHAQUE action utilisateur identifiee en Phase 1 :

```
Prompt :

Tu es un auditeur de flux fonctionnels. Pour le projet a [chemin], trace chaque
action utilisateur de bout en bout.

POUR CHAQUE BOUTON / FORMULAIRE / INTERACTION dans l'app :

1. ELEMENT UI — Identifier dans le JSX :
   - Quel composant ? Quel fichier:ligne ?
   - A-t-il un handler (onClick, onSubmit, onChange) ?
   - Si non -> RUPTURE : "Bouton sans handler"

2. HANDLER — Suivre le handler :
   - Que fait-il ? (appel API, setState, navigation, etc.)
   - Appelle-t-il une API route ? Laquelle ?
   - Si appel API : URL correcte ? Methode correcte ? Body envoye ?
   - Si pas d'appel API alors qu'il devrait y en avoir -> RUPTURE
   - Gere-t-il le loading state ?
   - Gere-t-il les erreurs (try/catch + feedback) ?

3. API ROUTE — Suivre la route appelee :
   - Le fichier existe-t-il ? A la bonne URL ?
   - Accepte-t-il la methode utilisee (POST/GET/etc.) ?
   - Valide-t-il les inputs recus ?
   - Appelle-t-il le bon service externe ?
   - Retourne-t-il une reponse structuree ?
   - Gere-t-il les erreurs (try/catch, codes HTTP) ?

4. SERVICE EXTERNE — Suivre l'appel au service :
   - Le client/SDK est-il initialise ?
   - La variable d'env est-elle definie ?
   - L'appel utilise-t-il les bons parametres ?
   - Si BDD : la table/collection existe-t-elle dans le schema ?
   - Si email : le template est-il defini ? Les champs dynamiques remplis ?
   - Si paiement : le flow est-il complet (intent -> confirmation) ?

5. RETOUR — Suivre le retour vers l'UI :
   - La reponse API est-elle traitee cote client ?
   - Le succes affiche-t-il un feedback (toast, message, redirect) ?
   - L'erreur affiche-t-elle un message comprehensible ?
   - L'UI se met-elle a jour (revalidation, state update) ?

FORMAT DE SORTIE — Un tableau par flux :

## Flux : [Nom du flux] (ex: "Soumission formulaire de contact")

| Etape | Fichier:Ligne | Statut | Detail |
|-------|---------------|--------|--------|
| UI Element | ContactForm.tsx:42 | OK | Bouton "Envoyer" avec onSubmit |
| Handler | ContactForm.tsx:15 | OK | handleSubmit() appelle /api/contact |
| Loading state | ContactForm.tsx:18 | MANQUANT | Pas de setLoading(true) |
| Appel API | ContactForm.tsx:20 | OK | fetch('/api/contact', { method: 'POST' }) |
| API Route | app/api/contact/route.ts:1 | OK | Fichier existe |
| Validation input | app/api/contact/route.ts:8 | MANQUANT | Pas de validation du body |
| Service externe | app/api/contact/route.ts:15 | RUPTURE | Appel Brevo mais BREVO_API_KEY non definie |
| Reponse API | app/api/contact/route.ts:25 | OK | Return NextResponse.json |
| Feedback succes | ContactForm.tsx:24 | MANQUANT | Pas de message de succes |
| Feedback erreur | ContactForm.tsx:28 | MANQUANT | catch vide |

STATUTS :
- OK : Fonctionne correctement
- MANQUANT : Le maillon n'existe pas du tout
- RUPTURE : Le maillon existe mais est casse (mauvaise URL, variable manquante, etc.)
- INCOMPLET : Le maillon existe mais partiellement (validation partielle, etc.)
- DECONNECTE : Le maillon existe mais n'est pas relie au reste du flux

A la fin, un resume :
### Resume des ruptures
| # | Flux | Etape cassee | Severite | Description |
|...|...|...|...|...|
```

### Agent 2 : Coherence des donnees & types

```
Prompt :

Tu es un auditeur de coherence de donnees. Pour le projet a [chemin],
verifie que les types, les schemas et les donnees sont coherents de bout en bout.

1. TYPES FRONTEND vs API :
   - Pour chaque appel fetch/axios cote client, identifier le type attendu en reponse
   - Pour chaque API route, identifier le type retourne
   - Verifier que les deux matchent
   - Verifier que les champs utilises dans le JSX existent dans le type

2. TYPES API vs BDD :
   - Pour chaque modele/schema BDD (Prisma, Mongoose, Supabase types, etc.)
   - Verifier que les API routes utilisent les bons champs
   - Verifier que les queries retournent les bons champs
   - Verifier que les mutations envoient les bons champs

3. FORMULAIRES vs API :
   - Pour chaque formulaire, lister les champs du form (name/id)
   - Verifier que le body envoye a l'API contient exactement ces champs
   - Verifier que l'API route attend exactement ces champs
   - Verifier que la validation cote client matche la validation cote API

4. PROPS vs USAGE :
   - Pour chaque composant avec des props typees (interface/type)
   - Verifier que le parent passe bien toutes les props requises
   - Verifier que le composant utilise bien toutes les props recues
   - Signaler les props optionnelles jamais passees (code mort)

5. ENV VARS :
   - Grep TOUT `process.env.XXX` dans le code
   - Verifier presence dans .env.local ou .env
   - Verifier que les NEXT_PUBLIC_ sont utilisees cote client et les autres cote serveur

FORMAT :
## Coherence Donnees & Types

### Desynchronisations Frontend <-> API
| # | Frontend (fichier:ligne) | API Route | Probleme |
|...|...|...|...|

### Desynchronisations API <-> BDD
| # | API Route | Schema/Modele | Probleme |
|...|...|...|...|

### Formulaires desynchronises
| # | Formulaire | Champs form | Champs API | Diff |
|...|...|...|...|...|

### Variables d'environnement
| # | Variable | Utilisee dans | Definie dans .env | Statut |
|...|...|...|...|...|
```

### Agent 3 : Etats UI & UX completude

```
Prompt :

Tu es un auditeur d'experience utilisateur. Pour le projet a [chemin],
verifie que chaque interaction a TOUS ses etats visuels geres.

1. ETATS DE CHARGEMENT — Pour chaque appel API/async dans le code :
   - Y a-t-il un loading state (useState, isLoading, isPending) ?
   - L'UI affiche-t-elle un indicateur (spinner, skeleton, disabled button) ?
   - Le bouton est-il desactive pendant le chargement ?
   - Si non -> l'utilisateur peut re-cliquer = double soumission

2. ETATS D'ERREUR — Pour chaque appel API/async :
   - Y a-t-il un try/catch ou .catch() ?
   - L'erreur est-elle affichee a l'utilisateur (toast, message inline, alert) ?
   - Le message est-il comprehensible (pas juste "Error" ou console.log) ?
   - Si erreur reseau -> y a-t-il un retry ou un message "verifiez votre connexion" ?

3. ETATS VIDES — Pour chaque liste/tableau/grille de donnees :
   - Que se passe-t-il quand il n'y a aucune donnee ?
   - Y a-t-il un empty state (message, illustration) ?
   - Ou est-ce juste un ecran blanc ?

4. ETATS DE SUCCES — Pour chaque formulaire/action :
   - Y a-t-il un feedback de succes (toast, message, redirect, animation) ?
   - Le formulaire se reset-il apres succes ?
   - L'utilisateur sait-il que son action a fonctionne ?

5. NAVIGATION & LIENS — Completude :
   - Chaque lien dans le header/footer/nav pointe vers une page qui existe
   - Le menu mobile fonctionne (toggle open/close, liens cliquables)
   - Le logo ramene a l'accueil
   - Les breadcrumbs sont corrects (si presents)
   - Active state sur la page courante dans la nav

6. RESPONSIVE — Verification :
   - Les composants utilisent-ils des classes responsive (sm:/md:/lg:) ?
   - Le menu mobile est-il different du desktop ?
   - Les grilles s'adaptent-elles (grid-cols-1 sur mobile) ?
   - Les textes ne debordent-ils pas (overflow, text truncation) ?

FORMAT :
## Etats UI & UX

### Loading states manquants
| # | Fichier:Ligne | Action | Consequence |
|...|...|...|...|

### Gestion d'erreurs manquante
| # | Fichier:Ligne | Appel API | Consequence |
|...|...|...|...|

### Empty states manquants
| # | Fichier:Ligne | Liste/Donnees | Consequence |
|...|...|...|...|

### Feedback succes manquant
| # | Fichier:Ligne | Action | Consequence |
|...|...|...|...|

### Navigation cassee
| # | Source | Lien/Destination | Probleme |
|...|...|...|...|
```

---

## Phase 3 : Rapport consolide

L'agent principal assemble les 3 rapports en un seul document.

### Format

```markdown
# Audit Fonctionnel — [Nom du projet]
**Date** : [date]
**Stack** : [stack detecte]
**Pages** : X | **API Routes** : X | **Actions utilisateur** : X

---

## Carte des flux

Pour chaque flux majeur de l'app, un schema simplifie :

### [Nom du flux]
UI (fichier) -> Handler (fichier:ligne) -> API (/api/xxx) -> Service (Brevo/Supabase/etc.) -> Retour UI
Statut : [COMPLET / CASSE / INCOMPLET]
Rupture(s) : [description si casse]

---

## Ruptures critiques (l'app ne fonctionne pas)

Ce qui est CASSE et empeche l'utilisateur de completer une action :

| # | Flux | Maillon casse | Fichier:Ligne | Description | Fix |
|---|------|---------------|---------------|-------------|-----|
| 1 | Contact | API -> Brevo | api/contact/route.ts:15 | BREVO_API_KEY non definie | Ajouter dans .env.local |
| 2 | Reservation | UI -> API | ReservationForm.tsx:42 | onSubmit vide | Implementer handleSubmit |
| ... | | | | | |

## Chainons manquants (feature incomplete)

Ce qui MANQUE pour que le flux soit complet :

| # | Flux | Maillon manquant | Ou l'ajouter | Description |
|---|------|------------------|--------------|-------------|
| 1 | Contact | Loading state | ContactForm.tsx | Pas de spinner pendant l'envoi |
| 2 | Contact | Feedback succes | ContactForm.tsx | Aucun message apres envoi |
| ... | | | | |

## Desynchronisations (les donnees ne matchent pas)

| # | Cote A | Cote B | Probleme |
|---|--------|--------|----------|
| 1 | Form envoie { name, email } | API attend { nom, courriel } | Noms de champs differents |
| ... | | | |

## Etats UI manquants

| # | Composant | Etat manquant | Impact utilisateur |
|---|-----------|---------------|-------------------|
| 1 | ReservationForm | Erreur | Echec silencieux |
| 2 | BlogList | Vide | Page blanche si 0 articles |
| ... | | | |

---

## Score de completude fonctionnelle

| Flux | UI | Handler | API | Service | Retour | Score |
|------|----|---------|-----|---------|--------|-------|
| Contact | OK | OK | INCOMPLET | RUPTURE | MANQUANT | 2/5 |
| Reservation | OK | MANQUANT | MANQUANT | MANQUANT | MANQUANT | 1/5 |
| Navigation | OK | OK | N/A | N/A | OK | 5/5 |
| ... | | | | | | |

**Score global : X/Y flux complets**
```

---

## Phase 4 : Corrections

### Priorisation stricte

1. **RUPTURES** — Flux casses qui empechent l'app de fonctionner
2. **CHAINONS MANQUANTS CRITIQUES** — Features dont le flux est >50% present mais incomplet
3. **DESYNCHRONISATIONS** — Les donnees ne circulent pas correctement
4. **ETATS UI** — Loading, erreur, succes, vide

### Methode de correction

Pour chaque rupture/chainon manquant :

1. **Lire tout le flux** — Comprendre l'intention du code existant
2. **Identifier le pattern du projet** — Comment les autres flux similaires fonctionnent ?
3. **Completer le maillon manquant** en suivant le meme pattern
4. **Verifier le flux complet** apres correction

### Regles

- **Suivre les patterns existants du projet** : si le projet utilise fetch -> utiliser fetch, pas axios
- **Ne rien inventer** : si un texte/contenu est necessaire et absent, mettre un TODO explicite
- **Completer, jamais supprimer** : une feature incomplete doit etre terminee, pas retiree
- **Corrections < 20 lignes** : faire directement avec Edit
- **Corrections multi-fichiers ou complexes** : deleguer a un sub-agent agent-dev
- **Verifier le build** apres chaque batch : `npx tsc --noEmit`

### Verification finale

1. `npx tsc --noEmit` — Zero erreur TypeScript
2. `npm run build` — Build reussi
3. Re-tracer chaque flux corrige pour confirmer la chaine complete

---

## Checklists par systeme

### Reservation en ligne
```
FLUX : Selection creneau -> Formulaire -> Soumission -> Confirmation

[ ] Calendrier/liste affiche les disponibilites
    UI: composant calendrier -> Donnees: source des creneaux (API/statique/BDD)
[ ] Selection d'une date/creneau met a jour l'affichage
    UI: onClick/onChange -> State: selectedDate/selectedSlot
[ ] Formulaire de reservation visible apres selection
    UI: condition d'affichage basee sur selectedSlot
[ ] Validation des champs du formulaire (nom, email, telephone, etc.)
    UI: validation HTML5 + JS -> Affichage erreurs inline
[ ] Soumission envoie les donnees a l'API
    Handler: onSubmit -> fetch('/api/reservations', { method: 'POST', body })
[ ] API valide les donnees recues
    API: zod/joi validation du body
[ ] API verifie que le creneau est encore disponible
    API: query BDD pour conflits
[ ] API cree la reservation en BDD
    API: INSERT/create dans la table reservations
[ ] API envoie un email de confirmation (si applicable)
    API: appel Brevo/Resend avec template + donnees dynamiques
[ ] API retourne une reponse succes avec details reservation
    API: return { success: true, reservation: {...} }
[ ] UI affiche une confirmation a l'utilisateur
    UI: message succes / redirect vers page confirmation
[ ] UI gere l'erreur si le creneau est pris
    UI: message "Ce creneau n'est plus disponible"
[ ] UI gere l'erreur reseau
    UI: message "Erreur de connexion, reessayez"
[ ] Bouton desactive pendant la soumission
    UI: disabled={isLoading} + spinner
```

### Formulaire de contact
```
FLUX : Remplissage -> Validation -> Envoi -> Confirmation

[ ] Champs presents : nom, email, telephone (opt), message
    UI: <input> et <textarea> avec name/id corrects
[ ] Validation client : email format, message non vide
    UI: pattern email + required + messages d'erreur custom
[ ] onSubmit appelle /api/contact en POST
    Handler: fetch('/api/contact', { method: 'POST', body: JSON.stringify(formData) })
[ ] API valide les champs (cote serveur aussi)
    API: verification email, message non vide, sanitization
[ ] API envoie l'email via service (Brevo, Resend, etc.)
    API: appel SDK avec from/to/subject/html
[ ] Variable d'env du service email definie
    ENV: BREVO_API_KEY ou RESEND_API_KEY dans .env.local
[ ] API retourne succes/erreur
    API: return NextResponse.json({ success: true/false })
[ ] UI affiche message de succes
    UI: setState ou toast "Message envoye !"
[ ] UI reset le formulaire apres succes
    UI: reset() ou setState des champs a ''
[ ] UI affiche message d'erreur si echec
    UI: catch -> message "Erreur lors de l'envoi"
[ ] Bouton desactive pendant l'envoi
    UI: disabled={isLoading}
```

### Authentification
```
FLUX : Login/Register -> Validation -> Auth -> Session -> Redirect

[ ] Formulaire login : email + password
[ ] Formulaire register : nom + email + password + confirmation
[ ] Validation client des champs
[ ] Appel API/service d'auth (NextAuth, Supabase, Clerk)
[ ] Gestion session/token apres auth reussie
[ ] Redirect vers page protegee apres login
[ ] Redirect vers login si non authentifie (middleware)
[ ] Logout deconnecte et redirect
[ ] Reset password : formulaire -> email -> lien -> nouveau mot de passe
[ ] Feedback erreur : "Email ou mot de passe incorrect"
[ ] Protection des routes API (verifier session)
```

### E-commerce
```
FLUX : Catalogue -> Panier -> Checkout -> Paiement -> Confirmation

[ ] Bouton "Ajouter au panier" appelle une fonction/API
[ ] Le panier se met a jour (state global/context/localStorage)
[ ] Page panier affiche les articles avec quantite modifiable
[ ] Suppression d'article fonctionne
[ ] Total calcule correctement (prix x quantite + frais)
[ ] Bouton "Commander" -> page checkout
[ ] Formulaire adresse de livraison
[ ] Selection mode de livraison
[ ] Integration paiement (Stripe Checkout / Elements)
[ ] Webhook ou callback confirme le paiement
[ ] Commande creee en BDD apres paiement confirme
[ ] Email de confirmation envoye
[ ] Page de confirmation affichee
[ ] Gestion echec paiement (message + retry)
```

---

## Autonomie

- **Phases 1-3** : Lancer automatiquement, sans demander
- **Phase 4** : Presenter le rapport et demander "Je corrige les X ruptures trouvees ?" avant d'agir
- **Si --report-only** : s'arreter apres le rapport
