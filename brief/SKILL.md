---
name: brief
description: Brief interactif ultra-detaille d'un projet web. Pose des questions structurees pour collecter TOUS les besoins avant de coder. Pour chaque fonctionnalite (reservation, paiement, backoffice...) creuse en profondeur chaque flux, chaque ecran, chaque etat. Genere un project-brief.md qui sert de source unique de verite. Zero improvisation. Usage /brief
metadata:
  author: FORGITWEB
  version: "2.0"
---

# Brief Projet Web

Interview structuree qui collecte tout ce dont Claude Code a besoin pour developper un site sans rien inventer.

## Principe fondamental

**Avant CHAQUE bloc d'information, TOUJOURS demander :**

> "Tu as deja [ce document/cette info] ? Si oui, donne-moi le fichier ou colle le contenu. Sinon je le genere pour toi."

**JAMAIS generer quelque chose que le client a deja produit.**
**JAMAIS inventer une info que le client n'a pas fournie.**

---

## Declenchement

- `/brief`
- "nouveau projet client"
- "on commence un site"
- "j'ai un nouveau client"
- "on fait le brief"

---

## Workflow

### Phase 0 : Accueil

```
"Salut ! On va construire le brief complet de ton projet.

Je vais te poser des questions bloc par bloc. A chaque etape,
dis-moi si tu as deja le document/l'info ou si c'est a moi
de le generer.

On commence ?"
```

---

### Phase 1 : Identite du projet (score /100, seuil 90+)

Poser les questions par groupes de 2-3 max via `AskUserQuestion`.
Apres chaque reponse, mettre a jour le score et montrer la progression.

#### Bloc 1.1 — Le client (20 pts)

**Demander :**
- Nom de l'entreprise / marque
- Secteur d'activite
- Localisation (ville, zone d'intervention)
- Personne de contact (nom, email, telephone)

```
Score Identite : [X]/20
```

#### Bloc 1.2 — L'objectif (25 pts)

**Demander :**
- Pourquoi ce site ? (vitrine, conversion, reservation, e-commerce...)
- Quel est le probleme actuel ? (pas de site, site obsolete, pas visible sur Google...)
- C'est quoi le succes pour le client ? (plus d'appels, plus de devis, vente en ligne...)
- Y a-t-il un site existant ? Si oui, URL.

```
Score Objectif : [X]/25
```

#### Bloc 1.3 — La cible (15 pts)

**Demander :**
- Qui sont les clients du client ? (particuliers, pros, age, localisation)
- Comment ils cherchent ce service ? (Google, bouche a oreille, reseaux sociaux)
- Quels mots-cles ils taperaient sur Google ?

```
Score Cible : [X]/15
```

#### Bloc 1.4 — La concurrence (10 pts)

**Demander :**
- 2-3 sites concurrents a regarder ?
- Qu'est-ce qui differencie le client de ses concurrents ?

```
Score Concurrence : [X]/10
```

#### Bloc 1.5 — Le budget & delais (10 pts)

**Demander :**
- Budget indicatif (fourchette)
- Deadline souhaitee
- Hebergement prevu (Vercel, autre)
- Nom de domaine existant ?

```
Score Budget : [X]/10
```

#### Bloc 1.6 — Contraintes techniques (20 pts)

**Demander :**
- Fonctionnalites obligatoires ? (formulaire contact, reservation, paiement, blog, backoffice...)
- Services tiers deja utilises ? (Google Analytics, CRM, Brevo, Stripe, Calendly...)
- Langues du site ? (FR seul, multilingue)
- Le client gere du contenu lui-meme ? (CMS necessaire ou pas)

```
Score Technique : [X]/20
```

#### Evaluation

```
Score Global Phase 1 : [X]/100

Ventilation :
- Identite client    : [X]/20
- Objectif           : [X]/25
- Cible              : [X]/15
- Concurrence        : [X]/10
- Budget & delais    : [X]/10
- Contraintes tech   : [X]/20

[Si < 90] : Il me manque des infos sur [bloc le plus faible]...
[Si >= 90] : Parfait, on passe au detail des fonctionnalites !
```

**Si score < 90 :** Poser des questions ciblees sur le bloc le plus faible. Iterer jusqu'a 90+.

---

### Phase 2 : Deep-dive Fonctionnalites

**REGLE CRITIQUE : Pour CHAQUE fonctionnalite identifiee en Phase 1 (bloc 1.6), ouvrir un sous-questionnaire dedie qui creuse TOUS les details.**

Le but : qu'il n'y ait AUCUNE zone d'ombre sur le fonctionnement de chaque feature. Claude Code doit pouvoir implementer chaque flux sans se poser une seule question.

#### Methode

Pour chaque fonctionnalite, suivre ce schema :

```
"On va detailler [NOM DE LA FEATURE].
Tu as deja un cahier des charges ou des specs pour ca ?
→ Si oui : donne-moi le document
→ Si non : je vais te poser des questions pour tout cadrer"
```

Si pas de document fourni, poser les questions suivantes (adapter selon la feature) :

#### Template de questions par fonctionnalite

**1. Flux utilisateur**
- Quel est le parcours complet ? (etape par etape, du debut a la fin)
- Quels ecrans/pages sont impliques ?
- Que voit l'utilisateur a chaque etape ?
- Comment il passe d'une etape a l'autre ?

**2. Donnees**
- Quelles infos l'utilisateur doit saisir ? (champs de formulaire, un par un)
- Quels champs sont obligatoires vs optionnels ?
- Y a-t-il des choix predeterminés ? (dropdown, radio, checkbox — lister les options)
- D'ou viennent les donnees affichees ? (BDD, API externe, fichier statique, saisie admin)

**3. Regles metier**
- Quelles sont les conditions/restrictions ? (horaires, capacite, tarifs, zones...)
- Y a-t-il des calculs automatiques ? (prix, duree, disponibilite...)
- Quelles validations ? (format email, date min/max, quantite max...)
- Y a-t-il des etats differents ? (en attente, confirme, annule, termine...)

**4. Notifications**
- Qui recoit un email/SMS a quelle etape ? (client, admin, les deux)
- Quel est le contenu de chaque notification ? (confirmation, rappel, annulation...)
- Via quel service ? (Brevo, Resend, autre)

**5. Admin / Backoffice**
- Le client gere cette feature comment ? (backoffice web, email, fichier, rien)
- Que peut modifier l'admin ? (prix, disponibilites, contenus, statuts...)
- Y a-t-il un dashboard ? Quelles donnees y apparaissent ?
- Quand l'admin modifie quelque chose, ca se repercute ou et comment ? (front en temps reel, au prochain build, manuellement)

**6. Cas limites**
- Que se passe-t-il si [cas problematique] ? (double reservation, paiement echoue, annulation tardive...)
- Y a-t-il un mode hors-ligne ou un fallback ?
- Que voit l'utilisateur quand il y a une erreur ?

**7. Integration technique**
- Quel service tiers ? (Stripe, Calendly, Google Calendar, API custom...)
- Qui a le compte / les cles API ?
- Y a-t-il deja une base de donnees ? (Supabase, Firebase, Prisma, aucune)

#### Exemples de deep-dive par feature type

##### Reservation en ligne
```
Questions specifiques :
- Qu'est-ce qu'on reserve ? (creneau horaire, prestation, table, chambre...)
- Quels sont les creneaux disponibles ? (jours, heures, duree)
- Qui definit les disponibilites ? (l'admin manuellement, un calendrier synchro, une API)
- Combien de reservations max par creneau ?
- Le client peut choisir une date + heure precise ou juste demander un RDV ?
- Y a-t-il un tarif affiche ? Modifiable par l'admin ?
- Le paiement est requis a la reservation ou sur place ?
- Si paiement en ligne : acompte ou totalite ?
- Email de confirmation : contenu exact ? (recapitulatif, adresse, lien annulation...)
- Le client peut modifier/annuler ? Jusqu'a quand ? Remboursement ?
- Rappel automatique avant le RDV ? (combien de temps avant)
- Le backoffice affiche quoi ? (liste des resas, calendrier, filtre par date/statut)
- L'admin peut bloquer des creneaux manuellement ?
- Si Google Calendar : synchro unidirectionnelle ou bidirectionnelle ?
```

##### Paiement en ligne
```
Questions specifiques :
- Quels produits/services sont payes ? (liste avec prix)
- Prix fixes ou variables ? Si variables, qui les definit ?
- TVA applicable ? Quel taux ?
- Stripe Connect ou Stripe Checkout ?
- Paiement unique ou recurrent (abonnement) ?
- Devise ? (EUR, multi-devises)
- Facture automatique ? Format et contenu ?
- Remboursement possible ? Conditions ?
- Que voit le client apres le paiement ? (page confirmation, email, les deux)
- Webhook Stripe : quels evenements ecouter ? (payment_intent.succeeded, charge.refunded...)
- Mode test avant mise en prod ?
```

##### Formulaire contact
```
Questions specifiques :
- Quels champs ? (nom, email, telephone, message, objet, fichier joint...)
- Choix d'objet : quelles options dans le dropdown ?
- Qui recoit l'email ? (quelle adresse)
- Email de confirmation au client ? (contenu)
- Anti-spam : honeypot, reCAPTCHA, ou autre ?
- Stockage des messages ? (juste email, ou aussi BDD)
- Service d'envoi : Brevo, Resend, autre ?
```

##### Blog / Actualites
```
Questions specifiques :
- Qui redige les articles ? (le client, nous, les deux)
- Combien d'articles au lancement ?
- Categories / tags ?
- CMS necessaire pour que le client publie seul ? Lequel ?
- Commentaires actives ?
- Partage reseaux sociaux ?
- Pagination ou scroll infini ?
```

##### Espace client / Authentification
```
Questions specifiques :
- Inscription libre ou sur invitation ?
- Methode d'auth : email/password, magic link, OAuth (Google, Facebook) ?
- Que voit le client dans son espace ? (historique commandes, reservations, profil...)
- Peut-il modifier ses infos ?
- Mot de passe oublie : quel flow ?
- Roles differents ? (client, admin, super-admin)
- Session : duree, remember me ?
```

##### E-commerce
```
Questions specifiques :
- Combien de produits ? Categories ?
- Gestion stock ? Alertes rupture ?
- Variations produit ? (taille, couleur, options)
- Panier : sauvegarde en session ou en BDD ?
- Livraison : zones, tarifs, transporteurs
- Code promo / reduction ?
- Avis clients sur les produits ?
- Commande : quels statuts ? (en attente, payee, expediee, livree, annulee)
- Emails transactionnels : a quelles etapes ?
- Page produit : quelles infos affichees ?
- Admin : comment ajouter/modifier un produit ?
```

#### Sortie Phase 2

Pour chaque fonctionnalite detaillee, produire une fiche structuree :

```markdown
### Feature : [Nom]

**Flux utilisateur :**
1. [etape 1] → page/ecran : [URL]
2. [etape 2] → ...

**Donnees :**
| Champ | Type | Obligatoire | Options/Validation |
|-------|------|------------|-------------------|
| [nom] | text | Oui | min 2 chars |
| [date] | date | Oui | >= aujourd'hui |

**Regles metier :**
- [regle 1]
- [regle 2]

**Notifications :**
| Evenement | Destinataire | Canal | Contenu |
|-----------|-------------|-------|---------|
| [evenement] | [qui] | [email/SMS] | [resume] |

**Admin :**
- [ce que l'admin peut faire]
- [comment ca se repercute en front]

**Cas limites :**
- [cas] → [comportement]

**Integration :**
- Service : [nom]
- Config : [details]
```

---

### Phase 3 : Contenus & Visuels

Pour CHAQUE element ci-dessous, demander AVANT de generer :

#### Bloc 3.1 — Arborescence / Pages

```
"Tu as deja une arborescence ou une liste de pages ?
(fichier, notes, brief SEO...)
→ Si oui : donne-moi le document
→ Si non : je te propose une arbo basee sur ce qu'on vient de definir"
```

Si fourni : lire le fichier (`textutil` pour RTF/DOCX, `Read` pour MD/TXT).
Si a generer : proposer une arbo et la faire valider.

#### Bloc 3.2 — Brief SEO / Mots-cles

```
"Tu as deja un brief SEO ou une etude de mots-cles ?
→ Si oui : donne-moi le fichier
→ Si non : je fais une recherche de mots-cles et je te propose un brief SEO"
```

Si fourni : lire et integrer.
Si a generer : utiliser `/seo-toolkit` ou `agent-seo` pour produire un brief.

#### Bloc 3.3 — Textes / Contenu redactionnel

```
"Tu as deja les textes des pages ? (fichier Word, RTF, Google Doc...)
→ Si oui : donne-moi le/les fichiers
→ Si non : on peut les rediger (agent-redacteur) une fois l'arbo validee"
```

Si fourni : lire avec `textutil` et integrer dans content.md.
Si a rediger : noter comme tache a faire apres validation de l'arbo.

#### Bloc 3.4 — Logo & Identite visuelle

```
"Tu as deja un logo et une charte graphique ?
(fichiers PNG/SVG, couleurs, polices...)
→ Si oui : donne-moi les fichiers
→ Si non : on definira une direction artistique (agent-da)"
```

Si fourni : noter les couleurs, polices, fichiers logo.
Si a creer : noter comme tache pour agent-da.

#### Bloc 3.5 — Maquettes / References visuelles

```
"Tu as des maquettes (Figma, images) ou des sites de reference
dont tu aimes le style ?
→ Si oui : donne-moi les liens/fichiers
→ Si non : on generera le design via notre workflow vibes"
```

Si fourni : integrer comme reference visuelle.
Si a creer : noter comme tache workflow vibes (design-system.md).

#### Bloc 3.6 — Photos / Images

```
"Tu as des photos professionnelles a utiliser ?
(photos du lieu, de l'equipe, des realisations...)
→ Si oui : ou sont-elles ? (dossier, lien)
→ Si non : on utilisera des images de qualite adaptees"
```

#### Bloc 3.7 — Mentions legales & RGPD

```
"Tu as deja les mentions legales ?
(SIRET, forme juridique, hebergeur...)
→ Si oui : donne-moi les infos
→ Si non : je les genererai avec les infos du client"
```

---

### Phase 4 : Generation du Brief

Une fois toutes les infos collectees, generer le fichier `project-brief.md` dans le dossier projet.

#### Structure du project-brief.md

```markdown
# Project Brief — [Nom du projet]

**Date** : [date]
**Client** : [nom]
**Contact** : [nom, email, telephone]
**Domaine** : [domaine.fr]
**Hebergement** : [Vercel / autre]

---

## 1. Objectif du site

**Type** : [vitrine / reservation / e-commerce / ...]
**Probleme actuel** : [description]
**Succes = ** : [KPI concrets]
**Site existant** : [URL ou "aucun"]

---

## 2. Cible

**Clientele** : [description]
**Zone geographique** : [ville + rayon]
**Comportement de recherche** : [comment ils trouvent le service]

---

## 3. Concurrence

| Concurrent | URL | Points forts | Points faibles |
|-----------|-----|-------------|---------------|
| [nom] | [url] | ... | ... |

**Differenciation client** : [ce qui le rend unique]

---

## 4. Arborescence

Source : [fournie par le client / generee]

| Page | URL | Objectif |
|------|-----|---------|
| Accueil | / | ... |
| [Page] | /slug | ... |

---

## 5. SEO

Source : [brief fourni / genere]

**Mots-cles principaux** : [liste]
**Mots-cles secondaires** : [liste]
**Strategie locale** : [oui/non, villes ciblees]

---

## 6. Contenu

Source : [textes fournis / a rediger]

**Fichiers fournis** :
- [fichier1.docx] → pages concernees

**A rediger** :
- [ ] [page1]

---

## 7. Identite visuelle

Source : [charte fournie / a creer]

**Logo** : [fichier(s)]
**Couleurs** :
- Primaire : [#HEX]
- Secondaire : [#HEX]
- Accent : [#HEX]
**Polices** : [noms]
**References visuelles** : [URLs ou fichiers]

---

## 8. Fonctionnalites — Vue d'ensemble

| Fonctionnalite | Priorite | Service tiers | Statut specs |
|----------------|---------|--------------|-------------|
| [feature] | Obligatoire | [Stripe/Brevo/...] | Detaillee |

---

## 9. Fonctionnalites — Specs detaillees

### 9.1 [Nom de la feature]

**Flux utilisateur :**
1. [etape] → [page/URL]
2. ...

**Donnees / Formulaire :**
| Champ | Type | Obligatoire | Validation / Options |
|-------|------|------------|---------------------|
| ... | ... | ... | ... |

**Regles metier :**
- [regle]

**Notifications :**
| Evenement | Destinataire | Canal | Contenu |
|-----------|-------------|-------|---------|
| ... | ... | ... | ... |

**Admin / Backoffice :**
- [actions possibles]
- [repercussion front]

**Cas limites :**
| Cas | Comportement attendu |
|-----|---------------------|
| ... | ... |

**Integration technique :**
- Service : [nom]
- Config : [details]
- BDD : [oui/non, schema]

---

### 9.2 [Feature suivante]

[Meme structure que 9.1]

---

## 10. Contraintes

- **Budget** : [fourchette]
- **Deadline** : [date]
- **Langues** : [FR / multi]
- **CMS** : [oui/non]
- **Stack** : Next.js 14+ App Router, TypeScript, Tailwind CSS, Vercel

---

## 11. Taches a realiser

### Fourni (pret)
- [x] [element fourni]

### A produire
- [ ] [element] → [agent responsable]

### Ordre de production
1. Brief SEO (si pas fourni)
2. Direction artistique (si pas fournie)
3. Design system (workflow vibes)
4. Redaction contenu (si pas fourni)
5. Developpement site
6. Tests & audit (full-audit + full-secu)
7. Deploiement

---

## 12. Notes & decisions

[Tout ce qui a ete decide pendant le brief]
```

---

### Phase 5 : Validation

Presenter un resume au format :

```
Brief genere : project-brief.md

Resume :
- [X] pages prevues
- [X] fonctionnalites detaillees
- [X] elements fournis / [X] a produire
- Chaque feature a son flux, ses champs, ses regles, ses cas limites
- Prochaine etape : [premiere tache a faire]

Tu veux modifier quelque chose avant qu'on commence ?
```

Attendre la validation avant de passer a la suite.

---

## Regles strictes

1. **JAMAIS inventer du contenu** — Si l'info n'est pas fournie et pas generee explicitement, laisser un placeholder `[A FOURNIR]`
2. **TOUJOURS demander avant de generer** — "Tu as deja X ?" avant chaque bloc
3. **TOUJOURS lire les fichiers fournis** — `textutil` pour RTF/DOCX, `Read` pour le reste
4. **Score 90+ obligatoire** en Phase 1 avant de passer a la Phase 2
5. **Deep-dive obligatoire** — Chaque fonctionnalite doit etre detaillee (flux, donnees, regles, notifs, admin, cas limites, integration). AUCUNE feature ne doit rester vague.
6. **2-3 questions max par tour** — Ne pas submerger
7. **Montrer la progression** — Score apres chaque bloc
8. **Le project-brief.md est la source unique de verite** — Tout le reste du workflow en decoule
9. **Si une reponse est vague, reformuler et reposer** — "Tu veux dire que [interpretation] ? Ou plutot [alternative] ?"
