---
name: full-audit
description: "Audit fonctionnel end-to-end d'un projet web. Trace chaque flux utilisateur du bouton jusqu'a la BDD/API : UI -> handler -> API route -> service externe/BDD -> retour UI. Detecte les chainons manquants, features non connectees. Teste en runtime via Playwright. Corrige tout. Usage : /full-audit [chemin-du-projet]"
compatibility: claude-code-only
---

# Full Audit — Audit fonctionnel end-to-end & Fix

Verifie que **chaque action utilisateur** dans l'app est connectee de bout en bout : du clic sur le bouton jusqu'a l'ecriture en BDD/envoi d'email/appel API, et retour au feedback utilisateur. Teste en runtime. Detecte tous les chainons manquants et les corrige.

## Principe fondamental

**L'audit ne regarde pas si le code est "propre". Il verifie si l'app MARCHE.**

Deux niveaux d'audit :
1. **Analyse statique** — Lire le code et tracer les flux sans executer
2. **Tests runtime** — Lancer l'app et tester pour de vrai avec Playwright

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

## Phase 0 : Build Check (Agent principal)

**Avant TOUT audit, verifier que le projet compile.** Si ca build pas, rien d'autre ne sert.

### Etapes

1. **Verifier les dependances** :
   ```bash
   cd [chemin] && npm ls --depth=0 2>&1
   ```
   - Si `node_modules` n'existe pas -> `npm install`
   - Si packages manquants -> les installer

2. **Verifier TypeScript** :
   ```bash
   npx tsc --noEmit 2>&1
   ```
   - Collecter TOUTES les erreurs TS
   - Classer par severite (error vs warning)

3. **Verifier le build Next.js** :
   ```bash
   npx next build 2>&1
   ```
   - Si echec : lire l'erreur, identifier le fichier, corriger
   - Relancer jusqu'a build OK

4. **Verifier les variables d'env** :
   - Grep tout `process.env.XXX` et `import.meta.env.XXX` dans le code
   - Verifier presence dans `.env.local` ou `.env`
   - Lister les manquantes

### Resultat Phase 0

```
BUILD STATUS : [OK / FAIL]
Erreurs TS : X
Erreurs build : X
Env vars manquantes : [liste]
```

**Si FAIL** : Corriger les erreurs de build AVANT de passer a la Phase 1.
**Si OK** : Passer a la Phase 1.

---

## Phase 1 : Detection d'intention & Cartographie (Agent principal)

**Objectif** : Comprendre ce que l'app est CENSEE faire, puis cartographier ce qu'elle fait VRAIMENT.

### 1.1 — Deviner l'intention du projet

Lire ces fichiers pour comprendre le but de l'app :

```
SOURCES D'INTENTION (par ordre de fiabilite) :
1. README.md / README — Description du projet, features listees
2. package.json "description" / "name" — Nom et description
3. CLAUDE.md / docs/ — Documentation technique
4. Noms des routes :
   - /reservation, /booking -> systeme de reservation
   - /cart, /checkout, /product -> e-commerce
   - /login, /register, /dashboard -> app avec auth
   - /blog, /articles -> blog/CMS
   - /contact -> formulaire de contact
5. Noms des composants et fichiers :
   - ReservationForm, BookingCalendar -> reservation
   - CartProvider, CheckoutForm -> e-commerce
   - AuthContext, LoginForm -> authentification
6. Dependances dans package.json :
   - stripe -> paiement
   - @prisma/client, @supabase/supabase-js -> BDD
   - @brevo/*, resend, nodemailer -> emails
   - next-auth, @clerk/* -> authentification
   - react-calendar, date-fns -> calendrier/reservation
```

Construire la **Declaration d'intention** :

```markdown
## Ce que l'app est CENSEE faire

**Type** : [Site vitrine / App avec reservation / E-commerce / Dashboard / Blog / etc.]

**Features attendues** :
1. [Feature] — Source : [README / nom de route / dependance]
2. [Feature] — Source : [README / nom de route / dependance]
...

**Services externes attendus** :
- [Service] pour [usage] — Source : [package.json / .env / code]
...
```

### 1.2 — Cartographier la realite

Pour chaque page, route API, composant : voir Phase 1 du skill original.

```
Pour chaque page dans app/ ou pages/ :
  - URL de la route
  - Composant principal
  - Type (statique SSG / dynamique SSR / client)
  - Composants enfants utilises

Pour chaque fichier dans app/api/ ou pages/api/ :
  - Route, methodes, ce qu'elle fait
  - Services externes appeles

Pour chaque page, TOUTES les actions utilisateur :
  - Boutons, formulaires, liens, interactions

Pour chaque service/API/BDD :
  - Variable d'env requise et presente ?
  - Client/SDK initialise ?
```

### 1.3 — Ecart intention vs realite

Comparer les deux listes :

```markdown
## Ecart Intention vs Realite

| Feature attendue | Source | Statut dans le code | Details |
|------------------|--------|---------------------|---------|
| Reservation en ligne | README + /reservation route | PARTIEL | Formulaire present, API manquante |
| Envoi email contact | /api/contact + Brevo dep | CASSE | Route existe, BREVO_API_KEY manquante |
| Paiement Stripe | package.json stripe | NON IMPLEMENTE | Dep installee, zero code Stripe |
| Blog | /blog route | COMPLET | Listing + articles fonctionnels |

STATUTS :
- COMPLET : La feature fonctionne de bout en bout
- PARTIEL : Certains maillons existent, d'autres manquent
- CASSE : Les maillons existent mais la chaine est rompue
- NON IMPLEMENTE : La feature est prevue mais le code n'existe pas
- FANTOME : Du code existe mais aucune intention trouvee (code mort ?)
```

---

## Phase 2 : Audit statique des flux (4 Sub-Agents paralleles)

Lancer **4 agents en parallele** avec `Task(subagent_type: "general-purpose")`.

Passer a chaque agent :
- Le chemin du projet
- La declaration d'intention (Phase 1.1)
- La cartographie (Phase 1.2)
- Le tableau d'ecart (Phase 1.3)

### Agent 1 : Tracage des flux UI -> API -> Service -> Retour

**Le coeur de l'audit.** Pour CHAQUE action utilisateur :

```
Prompt :

Tu es un auditeur de flux fonctionnels. Pour le projet a [chemin], trace chaque
action utilisateur de bout en bout.

CONTEXTE :
[Coller la declaration d'intention + cartographie + ecart]

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

CONTEXTE :
[Coller la declaration d'intention + cartographie]

1. TYPES FRONTEND vs API :
   - Pour chaque appel fetch/axios cote client, identifier le type attendu en reponse
   - Pour chaque API route, identifier le type retourne
   - Verifier que les deux matchent
   - Verifier que les champs utilises dans le JSX existent dans le type

2. TYPES API vs BDD/SCHEMA :
   - Trouver le schema BDD : prisma/schema.prisma, types Supabase, mongoose models, etc.
   - Pour chaque modele/table, lister les champs
   - Verifier que les API routes utilisent les bons noms de champs
   - Verifier que les queries retournent les bons champs
   - Verifier que les mutations envoient les bons champs
   - Signaler les champs du schema jamais utilises (potentiellement inutiles)
   - Signaler les champs utilises dans le code mais absents du schema

3. FORMULAIRES vs API :
   - Pour chaque formulaire, lister les champs du form (name/id)
   - Verifier que le body envoye a l'API contient exactement ces champs
   - Verifier que l'API route attend exactement ces champs
   - Verifier que la validation cote client matche la validation cote API

4. PROPS vs USAGE :
   - Pour chaque composant avec des props typees (interface/type)
   - Verifier que le parent passe bien toutes les props requises
   - Verifier que le composant utilise bien toutes les props recues

5. ENV VARS :
   - Grep TOUT `process.env.XXX` dans le code
   - Verifier presence dans .env.local ou .env
   - Verifier que les NEXT_PUBLIC_ sont utilisees cote client et les autres cote serveur

FORMAT :
## Coherence Donnees & Types

### Desynchronisations Frontend <-> API
| # | Frontend (fichier:ligne) | API Route | Probleme |
|...|...|...|...|

### Desynchronisations API <-> BDD/Schema
| # | API Route | Schema/Modele | Champ | Probleme |
|...|...|...|...|...|

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

CONTEXTE :
[Coller la declaration d'intention + cartographie]

1. ETATS DE CHARGEMENT — Pour chaque appel API/async dans le code :
   - Y a-t-il un loading state (useState, isLoading, isPending) ?
   - L'UI affiche-t-elle un indicateur (spinner, skeleton, disabled button) ?
   - Le bouton est-il desactive pendant le chargement ?
   - Si non -> l'utilisateur peut re-cliquer = double soumission

2. ETATS D'ERREUR — Pour chaque appel API/async :
   - Y a-t-il un try/catch ou .catch() ?
   - L'erreur est-elle affichee a l'utilisateur (toast, message inline, alert) ?
   - Le message est-il comprehensible (pas juste "Error" ou console.log) ?

3. ETATS VIDES — Pour chaque liste/tableau/grille de donnees :
   - Que se passe-t-il quand il n'y a aucune donnee ?
   - Y a-t-il un empty state (message, illustration) ?

4. ETATS DE SUCCES — Pour chaque formulaire/action :
   - Y a-t-il un feedback de succes (toast, message, redirect, animation) ?
   - Le formulaire se reset-il apres succes ?

5. NAVIGATION & LIENS :
   - Chaque lien dans le header/footer/nav pointe vers une page qui existe
   - Le menu mobile fonctionne (toggle open/close, liens cliquables)
   - Le logo ramene a l'accueil
   - Active state sur la page courante dans la nav

6. RESPONSIVE :
   - Les composants utilisent-ils des classes responsive (sm:/md:/lg:) ?
   - Le menu mobile est-il different du desktop ?
   - Les grilles s'adaptent-elles ?

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

### Agent 4 : Synchronisation des donnees & Flux bidirectionnels

**Detecte quand une donnee modifiee quelque part (backoffice, admin, BDD) ne se repercute pas ailleurs (frontend, autre page, autre composant).**

C'est l'agent qui trouve les bugs du type : "je change un prix dans le backoffice mais ca change pas en front".

```
Prompt :

Tu es un auditeur de synchronisation de donnees. Pour le projet a [chemin],
verifie que quand une donnee est MODIFIEE quelque part, elle se met a jour PARTOUT
ou elle est affichee.

CONTEXTE :
[Coller la declaration d'intention + cartographie]

1. IDENTIFIER TOUTES LES SOURCES DE DONNEES :
   Pour chaque donnee affichee dans l'app (prix, nom, description, statut, date, etc.) :
   - D'ou vient-elle ? (BDD, API externe, fichier JSON local, hardcodee dans le code, .env)
   - Ou est-elle affichee ? (quelles pages, quels composants)
   - Ou est-elle modifiable ? (backoffice, admin, API, dashboard)

2. TRACER LES FLUX D'ECRITURE (modification) :
   Pour chaque formulaire/action qui MODIFIE une donnee :
   - Quel composant permet la modif ?
   - Quel handler est appele ?
   - Quelle API route est appelee ?
   - Comment la donnee est-elle ecrite ? (UPDATE BDD, PUT API, ecriture fichier)
   - Apres l'ecriture, y a-t-il un mecanisme de propagation ?

3. TRACER LES FLUX DE LECTURE (affichage) :
   Pour chaque endroit ou la donnee est AFFICHEE :
   - La donnee est-elle fetchee dynamiquement (API call a chaque visite) ?
   - Ou est-elle statique/cachee ? (SSG, ISR, cache React Query, localStorage)
   - Si cachee : quel est le mecanisme de revalidation ?
   - Si statique (SSG) : un rebuild est-il necessaire pour voir la modif ?

4. DETECTER LES RUPTURES DE SYNCHRONISATION :

   Type A — DONNEES HARDCODEES :
   La donnee est ecrite en dur dans le code (const prices = [...], const services = [...])
   alors qu'elle devrait venir de la BDD/API.
   -> Resultat : modifier en backoffice ne change RIEN en front.
   -> Exemples : prix dans un tableau JS, horaires en dur, liste de services statique

   Type B — CACHE SANS REVALIDATION :
   La donnee est fetchee mais cachee (ISR, React Query, SWR, localStorage)
   et il n'y a PAS de mecanisme pour invalider le cache apres modification.
   -> Resultat : la modif apparait apres X secondes/minutes, ou jamais.
   -> Verifier : revalidatePath(), revalidateTag(), mutate(), invalidateQueries()
   -> Verifier : les options de cache (next: { revalidate: X }, staleTime, cacheTime)
   -> Verifier : si SSG/ISR, y a-t-il un webhook ou on-demand revalidation apres modif ?

   Type C — SOURCES DIVERGENTES :
   Le backoffice et le frontend ne lisent PAS la meme source.
   -> Exemple : backoffice lit/ecrit dans table "services", front lit un fichier JSON genere au build
   -> Exemple : backoffice ecrit dans Supabase, front lit depuis un CMS headless different
   -> Exemple : backoffice modifie le champ "price", front affiche le champ "tarif" (autre champ)

   Type D — PROPAGATION MANQUANTE :
   L'ecriture en BDD fonctionne, mais aucun signal n'est envoye pour mettre a jour le front.
   -> Verifier : apres un PUT/PATCH, y a-t-il un revalidatePath/revalidateTag ?
   -> Verifier : apres une mutation, y a-t-il un router.refresh() ou un refetch ?
   -> Verifier : si temps reel necessaire, y a-t-il un websocket/SSE/polling ?

   Type E — ETAT LOCAL DESYNCHRONISE :
   Le composant utilise un state local (useState) initialise au mount,
   mais ne se re-synchronise jamais apres une modification externe.
   -> Exemple : prix charge dans useState au mount, jamais mis a jour
   -> Verifier : y a-t-il un useEffect avec dependency array qui refetch ?
   -> Verifier : le state est-il derive des props (qui elles se mettent a jour) ?

5. POUR CHAQUE DONNEE PARTAGEE (affichee ET modifiable), CONSTRUIRE LA MATRICE :

   Donnee : [ex: prix d'un service]
   Source de verite : [ex: table "services" dans Supabase]
   Modification : [ex: backoffice/admin/services -> PUT /api/services/:id -> Supabase update]
   Affichage : [ex: page /reservation -> GET /api/services -> liste des services avec prix]

   Apres modification, la donnee se met-elle a jour en front ?
   - [ ] L'API route de lecture lit la meme source que l'API route d'ecriture
   - [ ] Le cache est invalide apres ecriture (revalidatePath, revalidateTag, mutate)
   - [ ] Si SSG : on-demand revalidation configuree (ou ISR avec revalidate court)
   - [ ] Le composant front refetch apres navigation (pas de stale state)
   - [ ] Pas de donnees hardcodees qui court-circuitent la BDD

FORMAT DE SORTIE :

## Synchronisation des Donnees

### Donnees partagees (modifiables + affichees)
| # | Donnee | Source de verite | Ou modifiee | Ou affichee | Synchro | Probleme |
|---|--------|-----------------|-------------|-------------|---------|----------|
| 1 | Prix services | Supabase.services | Backoffice /admin | /reservation | ROMPUE | Front lit prix hardcodes |
| 2 | Horaires | fichier JSON | Backoffice /admin | Header + /contact | ROMPUE | JSON genere au build, pas de rebuild |
| 3 | Statut resa | Supabase.reservations | Backoffice | /admin/reservations | OK | Refetch a chaque visite |

### Ruptures de synchronisation detaillees
| # | Donnee | Type rupture | Flux ecriture | Flux lecture | Cause racine | Fix propose |
|---|--------|-------------|---------------|-------------|-------------|-------------|
| 1 | Prix | A (hardcode) | PUT /api/services -> Supabase OK | Front: const prices = [...] | Prix en dur dans ServiceCard.tsx:12 | Remplacer par fetch /api/services |
| 2 | Horaires | B (cache) | PUT /api/settings -> JSON OK | Front: import data from horaires.json | Fichier statique, pas de revalidation | ISR + revalidateTag('settings') |

### Donnees sans source dynamique (hardcodees)
| # | Donnee | Fichier:Ligne | Type | Devrait venir de |
|---|--------|---------------|------|-----------------|
| 1 | Prix: [50, 75, 120] | ServiceCard.tsx:12 | Array literal | BDD / API |
| 2 | Horaires: "9h-18h" | Footer.tsx:45 | String literal | BDD / API |
| 3 | Services: [{name:...}] | data/services.ts:1 | Fichier data | BDD / API |

### Strategie de cache
| Page/Composant | Type rendu | Cache | Revalidation | Suffisant ? |
|----------------|-----------|-------|--------------|-------------|
| /reservation | SSG | Build time | Aucune | NON (donnees dynamiques) |
| /admin | Client | React Query 5min | staleTime | OUI |
| /blog | ISR 3600s | 1h | revalidate | ACCEPTABLE |
```

---

## Phase 3 : Tests runtime avec Playwright

**Apres l'analyse statique, tester l'app en vrai.**

### 3.1 — Lancer le serveur de dev

```bash
pkill -f "next dev" 2>/dev/null
trash [chemin]/.next 2>/dev/null
npm --prefix [chemin] run dev > /tmp/audit-dev.log 2>&1 &
# Attendre "Ready" dans les logs
```

### 3.2 — Tests automatiques par page

Utiliser le skill `playwright-cli` pour chaque page :

```
Pour CHAQUE page de l'app :

1. SCREENSHOT — Prendre un screenshot de la page
   - Verifier visuellement que la page n'est pas cassee (pas d'ecran blanc, pas d'erreur affichee)
   - Verifier le rendu desktop ET mobile (viewport 375px)

2. CONSOLE ERRORS — Verifier la console du navigateur
   - Collecter TOUTES les erreurs console (errors, warnings)
   - Classifier : erreur JS, 404 resources, CORS, deprecation
   - Les erreurs JS = bugs a corriger

3. NETWORK — Verifier les appels reseau
   - Lister tous les appels API faits au chargement de la page
   - Y a-t-il des 404 ? Des 500 ? Des timeouts ?
   - Des appels qui ne devraient pas etre la ?
```

### 3.3 — Tests des flux interactifs

```
Pour CHAQUE formulaire :
  1. Remplir avec des donnees valides
  2. Soumettre
  3. Verifier : loading state apparait ?
  4. Verifier : reponse recue ? (succes ou erreur)
  5. Verifier : feedback affiche a l'utilisateur ?
  6. Verifier : erreurs console ?

Pour CHAQUE formulaire (test erreur) :
  1. Soumettre vide (sans remplir)
  2. Verifier : validation client bloque la soumission ?
  3. Remplir avec des donnees invalides (email sans @, etc.)
  4. Verifier : messages d'erreur affiches ?

Pour CHAQUE lien de navigation :
  1. Cliquer le lien
  2. Verifier : la page cible charge sans erreur ?
  3. Verifier : l'URL change correctement ?
  4. Verifier : pas d'erreur console ?

Pour le menu mobile :
  1. Viewport 375px
  2. Cliquer le burger
  3. Verifier : le menu s'ouvre ?
  4. Cliquer un lien du menu
  5. Verifier : navigation + menu se ferme ?

Pour CHAQUE interaction (calendrier, modal, dropdown, etc.) :
  1. Declencher l'interaction (clic, hover)
  2. Verifier : l'element reagit visuellement ?
  3. Verifier : l'etat se met a jour ?
  4. Verifier : pas d'erreur console ?
```

### 3.4 — Format des resultats runtime

```markdown
## Tests Runtime

### Pages testees
| # | Page | URL | Screenshot | Console errors | Network errors | Statut |
|---|------|-----|-----------|----------------|----------------|--------|
| 1 | Accueil | / | OK | 0 | 0 | OK |
| 2 | Reservation | /reservation | OK | 2 errors | 1x 404 | FAIL |
| 3 | Contact | /contact | OK | 0 | 0 | OK |

### Formulaires testes
| # | Formulaire | Page | Soumission valide | Soumission vide | Validation | Feedback | Statut |
|---|------------|------|-------------------|-----------------|------------|----------|--------|
| 1 | Contact | /contact | Timeout | Pas de blocage | MANQUANT | MANQUANT | FAIL |
| 2 | Reservation | /reservation | N/A (bouton desactive) | N/A | N/A | N/A | BLOQUE |

### Navigation testee
| # | Lien | Source | Destination | Resultat |
|---|------|--------|-------------|----------|
| 1 | "Reservation" dans nav | Header | /reservation | OK |
| 2 | "Blog" dans nav | Header | /blog | 404 |

### Erreurs console collectees
| # | Page | Type | Message | Fichier:Ligne |
|---|------|------|---------|---------------|
| 1 | /reservation | Error | "Cannot read properties of undefined" | ReservationForm.tsx:42 |
| 2 | /reservation | 404 | GET /api/slots 404 | - |
```

---

## Phase 4 : Rapport consolide avec correlation

L'agent principal assemble TOUS les resultats et les **correle entre eux**.

### 4.1 — Correlation des findings

Croiser les resultats des 4 agents statiques + les tests runtime :

```
EXEMPLE DE CORRELATION :
- Agent 1 trouve : "onSubmit du formulaire contact appelle /api/contact"
- Agent 2 trouve : "le form envoie { name, email } mais l'API attend { nom, courriel }"
- Agent 3 trouve : "pas de feedback erreur sur le formulaire contact"
- Runtime trouve : "soumission du formulaire contact -> erreur 400 en console"

CORRELATION : Le formulaire contact est CASSE.
  Cause racine : desynchronisation des noms de champs (Agent 2)
  Consequence : erreur 400 (Runtime)
  Aggravant : l'utilisateur ne voit rien car pas de feedback erreur (Agent 3)
  Fix : renommer les champs dans le fetch body OU dans l'API route
```

Regrouper les findings par **flux** et non par agent :

```markdown
## Flux : Formulaire de contact

### Analyse statique
| Etape | Statut | Detail | Source |
|-------|--------|--------|--------|
| UI Element | OK | Bouton avec onSubmit | Agent 1 |
| Handler -> API | RUPTURE | Noms de champs differents | Agent 2 |
| Feedback erreur | MANQUANT | Catch vide | Agent 3 |

### Test runtime
| Test | Resultat |
|------|----------|
| Soumission valide | Erreur 400 |
| Soumission vide | Pas de validation |
| Console | "400 Bad Request" |

### Diagnostic
**Cause racine** : Champs du formulaire nommes `name`/`email` mais API attend `nom`/`courriel`
**Impact** : Le formulaire de contact ne fonctionne PAS. Aucun message n'est jamais envoye.
**Fix** : Aligner les noms de champs (1 fichier, ~5 lignes)
```

### 4.2 — Schema de flux visuel (Mermaid)

Pour chaque flux majeur, generer un diagramme Mermaid :

```markdown
### Flux : Formulaire de contact

```mermaid
graph LR
    A[Bouton Envoyer] -->|onSubmit| B[handleSubmit]
    B -->|fetch POST| C[/api/contact]
    C -->|SDK| D[Brevo API]
    D -->|email| E[Boite mail client]
    C -->|response| F[Feedback UI]

    style A fill:#22c55e,color:#fff
    style B fill:#22c55e,color:#fff
    style C fill:#ef4444,color:#fff
    style D fill:#ef4444,color:#fff
    style F fill:#f59e0b,color:#fff

    %% Legende: vert=OK, rouge=RUPTURE, orange=MANQUANT
```

Couleurs :
- Vert (#22c55e) : OK
- Rouge (#ef4444) : RUPTURE
- Orange (#f59e0b) : MANQUANT
- Gris (#6b7280) : DECONNECTE
```

### 4.3 — Matrice de priorite par impact business

Classer chaque probleme par **impact business**, pas juste severite technique :

```markdown
## Matrice de priorite

| Impact business | Description | Exemples |
|-----------------|-------------|----------|
| BLOQUANT | L'utilisateur NE PEUT PAS utiliser une feature core | Reservation cassee, paiement en echec, formulaire contact ne fonctionne pas |
| DEGRADANT | L'utilisateur PEUT utiliser la feature mais avec friction | Pas de loading (double clic), pas de message d'erreur (echec silencieux) |
| COSMETIQUE | N'empeche pas l'utilisation mais degrade l'experience | Pas de empty state, active state manquant dans la nav |
| INVISIBLE | N'affecte pas l'utilisateur mais degrade la maintenabilite | Types desynchronises, props inutilisees, code mort |

### Problemes par priorite

#### BLOQUANT — A corriger IMMEDIATEMENT
| # | Flux | Probleme | Cause racine | Fichier(s) | Fix estime |
|---|------|----------|-------------|------------|------------|
| 1 | Contact | Formulaire ne fonctionne pas | Champs desynchronises | ContactForm.tsx, route.ts | 5 min |
| 2 | Reservation | API route manquante | /api/reservations n'existe pas | a creer | 30 min |

#### DEGRADANT — A corriger cette semaine
| # | Flux | Probleme | Fichier(s) |
|---|------|----------|------------|
| 1 | Contact | Pas de loading state | ContactForm.tsx |
| 2 | Contact | Pas de feedback succes | ContactForm.tsx |

#### COSMETIQUE — A corriger quand possible
...

#### INVISIBLE — A corriger lors du prochain refacto
...
```

### 4.4 — Format du rapport final complet

```markdown
# Audit Fonctionnel — [Nom du projet]
**Date** : [date]
**Stack** : [stack detecte]
**Pages** : X | **API Routes** : X | **Actions utilisateur** : X

---

## Ce que l'app est CENSEE faire
[Declaration d'intention de la Phase 1.1]

## Ecart intention vs realite
[Tableau de la Phase 1.3]

---

## Carte des flux (Mermaid)
[Un diagramme par flux majeur]

---

## Resultats par flux
[Pour chaque flux : analyse statique + runtime + diagnostic + cause racine]

---

## Matrice de priorite
[BLOQUANT / DEGRADANT / COSMETIQUE / INVISIBLE]

---

## Score de completude fonctionnelle

| Flux | UI | Handler | API | Service | Retour | Runtime | Score |
|------|----|---------|-----|---------|--------|---------|-------|
| Contact | OK | OK | RUPTURE | RUPTURE | MANQUANT | FAIL | 2/6 |
| Reservation | OK | MANQUANT | MANQUANT | MANQUANT | MANQUANT | FAIL | 1/6 |
| Navigation | OK | OK | N/A | N/A | OK | OK | 6/6 |

**Score global : X/Y flux complets**
**Problemes : X bloquants, X degradants, X cosmetiques**
```

---

## Phase 5 : Corrections

### Priorisation stricte (ordre business)

1. **BLOQUANT** — Les features core qui ne marchent pas du tout
2. **DEGRADANT** — Les features qui marchent mal (pas de loading, pas de feedback)
3. **DESYNCHRONISATIONS** — Les donnees qui ne circulent pas correctement
4. **COSMETIQUE** — Les etats UI manquants
5. **INVISIBLE** — Le cleanup technique

### Methode de correction

Pour chaque probleme :

1. **Lire la cause racine** identifiee dans le rapport correle
2. **Lire tout le flux** — Comprendre l'intention du code existant
3. **Identifier le pattern du projet** — Comment les autres flux similaires fonctionnent ?
4. **Corriger la cause racine** (pas le symptome)
5. **Verifier le flux complet** apres correction (re-test avec Playwright)

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
3. Relancer les tests Playwright sur chaque flux corrige
4. Confirmer : zero erreur console, formulaires fonctionnels, navigation OK
5. Re-generer les diagrammes Mermaid avec les nouveaux statuts (tout vert)

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

- **Phase 0** : Lancer automatiquement, corriger les erreurs de build sans demander
- **Phases 1-4** : Lancer automatiquement, produire le rapport
- **Phase 5** : Presenter le rapport et demander "Je corrige les X problemes bloquants ?" avant d'agir
- **Si --report-only** : s'arreter apres le rapport (Phase 4)
