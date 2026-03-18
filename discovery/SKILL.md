---
name: discovery
description: Discovery interactive d'un projet web. Pose des questions structurees pour collecter TOUS les besoins avant de coder. Genere un project-brief.md ultra-detaille qui sert de source unique de verite pour Claude Code. Zero improvisation possible. Usage /discovery
metadata:
  author: FORGITWEB
  version: "1.0"
---

# Discovery Projet Web

Interview structuree qui collecte tout ce dont Claude Code a besoin pour developper un site sans rien inventer.

## Principe fondamental

**Avant CHAQUE bloc d'information, TOUJOURS demander :**

> "Tu as deja [ce document/cette info] ? Si oui, donne-moi le fichier ou colle le contenu. Sinon je le genere pour toi."

**JAMAIS generer quelque chose que le client a deja produit.**
**JAMAIS inventer une info que le client n'a pas fournie.**

---

## Declenchement

- `/discovery`
- "nouveau projet client"
- "on commence un site"
- "j'ai un nouveau client"

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
Score Global Discovery : [X]/100

Ventilation :
- Identite client    : [X]/20
- Objectif           : [X]/25
- Cible              : [X]/15
- Concurrence        : [X]/10
- Budget & delais    : [X]/10
- Contraintes tech   : [X]/20

[Si < 90] : Il me manque des infos sur [bloc le plus faible]...
[Si >= 90] : Parfait, on passe aux contenus !
```

**Si score < 90 :** Poser des questions ciblees sur le bloc le plus faible. Iterer jusqu'a 90+.

---

### Phase 2 : Contenus & Visuels

Pour CHAQUE element ci-dessous, demander AVANT de generer :

#### Bloc 2.1 — Arborescence / Pages

```
"Tu as deja une arborescence ou une liste de pages ?
(fichier, notes, brief SEO...)
→ Si oui : donne-moi le document
→ Si non : je te propose une arbo basee sur ce qu'on vient de definir"
```

Si fourni : lire le fichier (`textutil` pour RTF/DOCX, `Read` pour MD/TXT).
Si a generer : proposer une arbo et la faire valider.

#### Bloc 2.2 — Brief SEO / Mots-cles

```
"Tu as deja un brief SEO ou une etude de mots-cles ?
→ Si oui : donne-moi le fichier
→ Si non : je fais une recherche de mots-cles et je te propose un brief SEO"
```

Si fourni : lire et integrer.
Si a generer : utiliser `/seo-toolkit` ou `agent-seo` pour produire un brief.

#### Bloc 2.3 — Textes / Contenu redactionnel

```
"Tu as deja les textes des pages ? (fichier Word, RTF, Google Doc...)
→ Si oui : donne-moi le/les fichiers
→ Si non : on peut les rediger (agent-redacteur) une fois l'arbo validee"
```

Si fourni : lire avec `textutil` et integrer dans content.md.
Si a rediger : noter comme tache a faire apres validation de l'arbo.

#### Bloc 2.4 — Logo & Identite visuelle

```
"Tu as deja un logo et une charte graphique ?
(fichiers PNG/SVG, couleurs, polices...)
→ Si oui : donne-moi les fichiers
→ Si non : on definira une direction artistique (agent-da)"
```

Si fourni : noter les couleurs, polices, fichiers logo.
Si a creer : noter comme tache pour agent-da.

#### Bloc 2.5 — Maquettes / References visuelles

```
"Tu as des maquettes (Figma, images) ou des sites de reference
dont tu aimes le style ?
→ Si oui : donne-moi les liens/fichiers
→ Si non : on generera le design via notre workflow vibes"
```

Si fourni : integrer comme reference visuelle.
Si a creer : noter comme tache workflow vibes (design-system.md).

#### Bloc 2.6 — Photos / Images

```
"Tu as des photos professionnelles a utiliser ?
(photos du lieu, de l'equipe, des realisations...)
→ Si oui : ou sont-elles ? (dossier, lien)
→ Si non : on utilisera des images de qualite adaptees"
```

#### Bloc 2.7 — Mentions legales & RGPD

```
"Tu as deja les mentions legales ?
(SIRET, forme juridique, hebergeur...)
→ Si oui : donne-moi les infos
→ Si non : je les genererai avec les infos du client"
```

---

### Phase 3 : Generation du Brief

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
- [fichier2.rtf] → pages concernees

**A rediger** :
- [ ] [page1]
- [ ] [page2]

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

## 8. Fonctionnalites

| Fonctionnalite | Priorite | Details |
|----------------|---------|---------|
| Formulaire contact | Obligatoire | Brevo API |
| [feature] | [Obligatoire/Souhaite/Optionnel] | [details] |

**Services tiers** :
- [service] : [utilisation]

---

## 9. Contraintes

- **Budget** : [fourchette]
- **Deadline** : [date]
- **Langues** : [FR / multi]
- **CMS** : [oui/non]
- **Stack** : Next.js 14+ App Router, TypeScript, Tailwind CSS, Vercel

---

## 10. Taches a realiser

### Fourni (pret)
- [x] [element fourni]

### A produire
- [ ] [Brief SEO] → agent-seo
- [ ] [Textes pages] → agent-redacteur
- [ ] [Direction artistique] → agent-da
- [ ] [Design system] → workflow vibes
- [ ] [Mentions legales] → generation auto
- [ ] [Code site] → agent-dev + Gemini MCP

### Ordre de production
1. Brief SEO (si pas fourni)
2. Direction artistique (si pas fournie)
3. Design system (workflow vibes)
4. Redaction contenu (si pas fourni)
5. Developpement site
6. Tests & audit (full-audit + full-secu)
7. Deploiement

---

## 11. Notes & decisions

[Tout ce qui a ete decide pendant la discovery]
```

---

### Phase 4 : Validation

Presenter un resume au format :

```
Brief genere : project-brief.md

Resume :
- [X] pages prevues
- [X] fonctionnalites
- [X] elements fournis / [X] a produire
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
5. **2-3 questions max par tour** — Ne pas submerger
6. **Montrer la progression** — Score apres chaque bloc
7. **Le project-brief.md est la source unique de verite** — Tout le reste du workflow en decoule
