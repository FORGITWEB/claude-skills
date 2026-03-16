---
name: full-secu
description: Audit de securite complet d'un projet web (Next.js, React, Node). Detecte les secrets exposes, failles XSS/injection, headers manquants, CORS mal configure, dependances vulnerables, API routes non protegees, env vars mal utilisees. Corrige automatiquement. Usage : /full-secu [chemin-du-projet]
metadata:
  author: FORGITWEB
  version: "1.0"
---

# Full Security Audit

Audit de securite complet d'un projet web. Fusionne detection de secrets, analyse de vulnerabilites code, headers HTTP, dependances, et configuration securite.

## Declenchement

Utiliser quand l'utilisateur demande :
- `/full-secu` ou `/full-secu chemin/projet`
- "verifie la securite de mon projet"
- "scan les failles de securite"
- "est-ce que mon site est securise"
- "cherche les cles API exposees"
- "audit securite"

## Workflow

### Phase 0 : Initialisation

```
1. Identifier le chemin du projet (argument ou demander)
2. Detecter la stack : Next.js / React / Node / autre
3. Lire package.json, .env*, .gitignore, next.config.*
4. Verifier que .env.local est dans .gitignore
```

### Phase 1 : 4 Agents en parallele

Lancer 4 agents `Task(subagent_type: "general-purpose")` EN PARALLELE :

---

#### Agent 1 : Secrets & Credentials Scanner

**Mission :** Trouver tout secret, cle API, token, mot de passe expose dans le code source.

**Patterns a detecter (regex) :**

```
AWS Access Key     : (A3T[A-Z0-9]|AKIA|AGPA|AIDA|AROA|AIPA|ANPA|ANVA|ASIA)[A-Z0-9]{16}
AWS Secret Key     : (?i)aws(.{0,20})?['\"][0-9a-zA-Z\/+]{40}['\"]
GitHub Token       : ghp_[0-9a-zA-Z]{36}
GitHub OAuth       : gho_[0-9a-zA-Z]{36}
Slack Webhook      : https://hooks\.slack\.com/services/T[a-zA-Z0-9_]{8,10}/B[a-zA-Z0-9_]{8,10}/[a-zA-Z0-9_]{24}
Private Key        : -----BEGIN (RSA|OPENSSH|DSA|EC|PGP) PRIVATE KEY-----
Generic API Key    : (?i)(api[_-]?key|apikey|access[_-]?key)(.{0,20})?['\"][0-9a-zA-Z]{32,}['\"]
Database URL       : (postgresql|mysql|mongodb):\/\/[^\s:]+:[^\s@]+@[^\s\/]+
JWT Token          : eyJ[a-zA-Z0-9_-]*\.eyJ[a-zA-Z0-9_-]*\.[a-zA-Z0-9_-]*
Brevo/Sendinblue   : xkeysib-[a-zA-Z0-9]{64}
Stripe Key         : (sk_live|pk_live|sk_test|pk_test)_[a-zA-Z0-9]{24,}
Google API Key     : AIza[0-9A-Za-z\-_]{35}
Firebase URL       : [a-z0-9-]+\.firebaseio\.com
Supabase Key       : eyJ[a-zA-Z0-9_-]{100,}
Vercel Token       : [a-zA-Z0-9]{24}
Generic Password   : (?i)(password|passwd|pwd|secret)[\s]*[=:][\s]*['\"][^'\"]{8,}['\"]
```

**Actions :**
1. Scanner tous les fichiers (sauf node_modules, .git, .next, dist, build)
2. Utiliser `Grep` avec chaque pattern
3. Ignorer les faux positifs : fichiers .example, .template, commentaires avec placeholder
4. Verifier que NEXT_PUBLIC_* ne contient PAS de cle secrete (Stripe secret, DB URL, etc.)
5. Verifier que les cles serveur (sans NEXT_PUBLIC_) ne sont pas importees cote client

**Format sortie :**
```
[CRITIQUE] src/lib/api.ts:15 — Cle Stripe sk_live_ en dur
[CRITIQUE] .env — Fichier .env non dans .gitignore
[ALERTE]   components/Map.tsx:8 — Google API Key en dur (devrait etre NEXT_PUBLIC_GOOGLE_MAPS_KEY)
[OK]       Aucune cle AWS detectee
```

---

#### Agent 2 : Vulnerabilites Code (XSS, Injection, CSRF)

**Mission :** Detecter les failles de securite dans le code applicatif.

**Checks XSS :**
- `dangerouslySetInnerHTML` sans sanitization (DOMPurify ou equivalent)
- Insertion de `searchParams` / `query` directement dans le JSX sans echappement
- `eval()`, `new Function()`, `setTimeout(string)` avec donnees dynamiques
- `document.write()`, `innerHTML` avec donnees utilisateur

**Checks Injection :**
- Requetes SQL construites par concatenation (pas de requetes parametrees)
- Commandes shell construites avec des variables utilisateur (`exec`, `spawn` sans validation)
- Path traversal : `fs.readFile(userInput)` sans validation de chemin
- LDAP injection, NoSQL injection patterns

**Checks CSRF :**
- API routes POST/PUT/DELETE sans verification CSRF token
- Formulaires sans token CSRF
- Cookies sans flag `SameSite`

**Checks Auth :**
- API routes sans middleware d'authentification
- JWT stocke dans localStorage (devrait etre httpOnly cookie)
- Comparaison de mot de passe non constant-time
- Token de session previsible

**Format sortie :**
```
[CRITIQUE] app/api/admin/route.ts — API route sans auth middleware
[CRITIQUE] components/Blog.tsx:42 — dangerouslySetInnerHTML sans sanitization
[ALERTE]   lib/db.ts:28 — Query SQL par concatenation
[ALERTE]   app/api/contact/route.ts — Pas de rate limiting
[INFO]     middleware.ts manquant — Pas de protection globale des routes
```

---

#### Agent 3 : Headers HTTP & Configuration Serveur

**Mission :** Verifier les headers de securite et la configuration du serveur.

**Headers a verifier dans next.config.* :**
```javascript
// Headers attendus
{
  'X-Content-Type-Options': 'nosniff',
  'X-Frame-Options': 'DENY',           // ou SAMEORIGIN si iframe necessaire
  'X-XSS-Protection': '1; mode=block',
  'Referrer-Policy': 'strict-origin-when-cross-origin',
  'Permissions-Policy': 'camera=(), microphone=(), geolocation=()',
  'Strict-Transport-Security': 'max-age=63072000; includeSubDomains; preload',
  'Content-Security-Policy': "default-src 'self'; ..."
}
```

**Checks CORS :**
- `Access-Control-Allow-Origin: *` sur des routes sensibles = CRITIQUE
- CORS trop permissif (wildcard methods, wildcard headers)
- Credentials + wildcard origin = interdit par spec mais parfois mal configure

**Checks next.config :**
- `poweredBy: false` (desactiver le header X-Powered-By)
- `images.remotePatterns` trop permissif
- Redirections HTTP → HTTPS
- `serverExternalPackages` avec des packages non necessaires

**Checks Cookies :**
- Flag `HttpOnly` sur les cookies de session
- Flag `Secure` en production
- `SameSite=Strict` ou `Lax`
- `Path` restreint

**Format sortie :**
```
[CRITIQUE] next.config — Aucun header de securite configure
[ALERTE]   next.config — X-Powered-By non desactive
[ALERTE]   API CORS — Access-Control-Allow-Origin: * sur /api/admin
[INFO]     CSP non configure — Recommande pour production
```

---

#### Agent 4 : Dependances & Configuration

**Mission :** Verifier les dependances vulnerables et la configuration globale de securite.

**Checks Dependances :**
```bash
# Lire le output de npm audit (ne pas executer, lire package-lock.json)
# Ou analyser package.json pour des packages connus vulnerables
```
- Packages avec CVE connues (verifier les versions dans package.json)
- Packages abandonnes (derniere mise a jour > 2 ans)
- Packages avec typosquatting potentiel
- `devDependencies` incluses en production

**Checks .gitignore :**
```
# Doit contenir au minimum :
.env
.env.local
.env.production
.env*.local
node_modules/
.next/
*.pem
*.key
.vercel
.DS_Store
```

**Checks Configuration :**
- TypeScript `strict: true` (previent certaines failles)
- ESLint avec regles securite (`eslint-plugin-security`)
- `.env.example` present (sans vraies valeurs)
- Pas de `console.log` avec des donnees sensibles en production
- `robots.txt` n'expose pas de routes admin
- `sitemap.xml` n'inclut pas de routes privees

**Checks Exposition :**
- Routes `/api/*` listees dans le code vs routes qui devraient etre privees
- Fichiers statiques dans `/public` contenant des infos sensibles
- Source maps actives en production (`productionBrowserSourceMaps: false`)
- `.env` ou `.env.local` accessible publiquement

**Format sortie :**
```
[CRITIQUE] .gitignore — .env.local manquant
[CRITIQUE] next.config — Source maps actives en production
[ALERTE]   package.json — next@14.0.1 a 3 CVE connues (mettre a jour vers 14.2+)
[ALERTE]   robots.txt — /admin et /api visibles
[INFO]     eslint-plugin-security non installe
```

---

### Phase 2 : Rapport Consolide

Fusionner les 4 rapports dans un format unifie :

```markdown
# Rapport Securite — [Nom du projet]
## Date : [date]
## Stack : [Next.js X.X / React X.X / Node X.X]

### Resume Executif
- X failles CRITIQUES (a corriger immediatement)
- X ALERTES (a corriger rapidement)
- X INFO (recommandations)
- Score global : X/100

### Failles CRITIQUES
1. [Description] — [Fichier:Ligne] — [Impact]
   Correction : [code ou instruction]

### ALERTES
1. ...

### Recommandations
1. ...

### Checklist Post-Correction
- [ ] Toutes les cles en .env.local
- [ ] .gitignore complet
- [ ] Headers securite configures
- [ ] API routes protegees
- [ ] Dependances a jour
- [ ] Source maps desactivees en prod
```

### Phase 3 : Corrections Automatiques

**Corriger automatiquement (avec confirmation utilisateur) :**

1. **Secrets exposes** → Deplacer dans .env.local, remplacer par `process.env.VAR_NAME`
2. **Headers manquants** → Ajouter le bloc headers dans next.config
3. **.gitignore incomplet** → Ajouter les entrees manquantes
4. **X-Powered-By** → Ajouter `poweredBy: false` dans next.config
5. **Source maps** → Ajouter `productionBrowserSourceMaps: false`
6. **dangerouslySetInnerHTML** → Wrapper avec DOMPurify

**Ne PAS corriger automatiquement (signaler uniquement) :**
- Dependances a mettre a jour (risque de breaking changes)
- CORS configuration (depend du use case)
- CSP (trop specifique au projet)
- Auth middleware (necessite comprehension du flow)

## Severite

| Niveau | Signification | Action |
|--------|--------------|--------|
| CRITIQUE | Exploitable immediatement, donnees en danger | Corriger AVANT deploiement |
| ALERTE | Risque reel mais exploitation moins directe | Corriger dans la semaine |
| INFO | Bonne pratique manquante, pas de risque immediat | Planifier |

## Exemples de Detection

### Exemple 1 : Cle API en dur
```typescript
// DETECTE :
const stripe = new Stripe('sk_live_abc123xyz789');

// CORRECTION :
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);
// + Ajouter STRIPE_SECRET_KEY=sk_live_abc123xyz789 dans .env.local
```

### Exemple 2 : API route non protegee
```typescript
// DETECTE : app/api/admin/users/route.ts
export async function GET() {
  const users = await db.user.findMany();
  return Response.json(users);
}

// SIGNALE : Cette route expose tous les utilisateurs sans authentification
// RECOMMANDATION : Ajouter middleware auth ou verification session
```

### Exemple 3 : XSS via dangerouslySetInnerHTML
```typescript
// DETECTE :
<div dangerouslySetInnerHTML={{ __html: post.content }} />

// CORRECTION :
import DOMPurify from 'dompurify';
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(post.content) }} />
```

### Exemple 4 : Headers manquants
```javascript
// CORRECTION next.config.js :
const nextConfig = {
  poweredBy: false,
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          { key: 'X-Frame-Options', value: 'DENY' },
          { key: 'X-XSS-Protection', value: '1; mode=block' },
          { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
          { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
        ],
      },
    ];
  },
};
```

## Contraintes

- Ne lance AUCUNE commande reseau (pas de scan nmap, pas de requete externe)
- Analyse uniquement le code source local
- Ne modifie rien sans confirmation utilisateur pour les corrections automatiques
- Les corrections de type "CRITIQUE secrets" peuvent etre appliquees directement
- Toujours `trash` au lieu de `rm` pour les fichiers a supprimer
