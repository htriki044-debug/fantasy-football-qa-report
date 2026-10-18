<div align="center">

# 🔷 Fantasy Football Web App — Rapport d'Investigation QA

**Analyse des causes racines, diagnostics techniques et correctifs suggérés**

![Type](https://img.shields.io/badge/Type-Rapport%20QA-1f6feb?style=for-the-badge&logo=github&logoColor=white)
![Bugs](https://img.shields.io/badge/Bugs%20analys%C3%A9s-2-1f6feb?style=for-the-badge)
![DevTools](https://img.shields.io/badge/Outils-Chrome%20DevTools-1f6feb?style=for-the-badge&logo=googlechrome&logoColor=white)
![Licence](https://img.shields.io/badge/Licence-MIT-1f6feb?style=for-the-badge)

</div>

---

## 🔹 Sommaire

1. [🔷 Présentation du projet](#-présentation-du-projet)
2. [🔷 Synthèse des bugs](#-synthèse-des-bugs)
3. [🔷 Bug 1 : Gel de l'UI lors de la navigation entre les journées](#-bug-1--gel-de-lui-lors-de-la-navigation-entre-les-journées)
4. [🔷 Bug 2 : Inversion domicile / extérieur des matchs](#-bug-2--inversion-domicile--extérieur-des-matchs)
5. [🔷 Compétences démontrées](#-compétences-démontrées)
6. [🔷 Licence](#-licence)

---

## 🔷 Présentation du projet

Ce dépôt est un projet de portfolio QA qui documente l'investigation de **deux bugs** découverts sur une application web de Fantasy Football.

Pour chaque bug, le rapport présente :

- 🔹 la **description du symptôme** observé ;
- 🔹 le **diagnostic technique** (analyse de la cause racine) ;
- 🔹 la **classification** (sévérité et priorité) ;
- 🔹 un **correctif suggéré**.

Les investigations s'appuient sur les **DevTools du navigateur** (onglets Console et Network), l'analyse des **payloads d'API**, des tests **cross-browser** et des principes de **programmation défensive**.

---

## 🔷 Synthèse des bugs

| 🔹 ID | Titre | Couche concernée | Sévérité | Priorité |
|:-----:|-------|:----------------:|:--------:|:--------:|
| **BUG-01** | Gel de l'UI lors de la navigation entre les journées | Front-end (client) | Haute | Basse |
| **BUG-02** | Inversion des équipes domicile / extérieur | Back-end (API / base de données) | Moyenne | Haute |

---

## 🔷 Bug 1 : Gel de l'UI lors de la navigation entre les journées

> 🔵 **Couche :** Front-end · **Sévérité :** Haute · **Priorité :** Basse

### 🔹 Présentation

Lors de la navigation entre les journées (Gameweeks) avec les boutons fléchés `<` et `>`, l'interface se fige complètement et ne répond plus. La communication réseau avec le serveur, elle, s'exécute correctement en arrière-plan.

### 🔹 Diagnostic technique

| 🔵 Source | Constat |
|-----------|---------|
| **Onglet Network** | Le clic déclenche une requête vers `/api/fixtures/?event=X`. Le serveur répond avec un statut `HTTP 200 OK` et le payload JSON attendu. |
| **Onglet Console** | Erreur `Failed to load resource: the server responded with a status of 403 ()` sur des scripts externes publicitaires et de suivi (`googletag` / `PubAdsService`). |
| **Cause racine** | Le rendu Front-end est étroitement couplé à l'exécution de ces scripts tiers. Lorsqu'un environnement client (bloqueur de publicités agressif, outil de traduction automatique du navigateur) bloque le script externe, le thread JavaScript principal plante **avant** le rafraîchissement du DOM. |

**Classification :** Sévérité **Haute** / Priorité **Basse**. La fonctionnalité clé est bloquée, mais seulement dans un contexte client spécifique.

### 🔹 Correctif suggéré (programmation défensive)

Le rendu de l'interface doit être découplé des scripts externes non critiques. Isoler le chargement de la dépendance externe dans un bloc `try...catch` dédié garantit que le DOM se met à jour même si le script de tracking échoue.

```javascript
// 🔷 Correctif proposé par la QA
async function handleGameweekNavigation(nextEventId) {
    try {
        // 🔵 1. Récupération sécurisée des données principales
        const response = await fetch(`/api/fixtures/?event=${nextEventId}`);
        if (!response.ok) throw new Error('Échec de la requête API');
        const data = await response.json();

        // 🔵 2. Rendu immédiat de l'UI pour garantir une expérience fluide
        updateGameweekDOM(data);

        // 🔵 3. Isolation de l'exécution du script tiers
        try {
            await loadExternalTrackingScript();
        } catch (resourceError) {
            // 🔹 Script externe bloqué (403) : le rendu de l'UI n'est pas affecté
            console.warn("Script externe bloqué (403), rendu de l'UI non affecté.", resourceError);
        }

    } catch (criticalError) {
        // 🔷 Erreur critique : informer l'utilisateur sans figer l'interface
        console.error("Erreur critique dans le thread de navigation de l'UI :", criticalError);
        showFriendlyErrorToUser();
    }
}
```

---

## 🔷 Bug 2 : Inversion domicile / extérieur des matchs

> 🔵 **Couche :** Back-end (API / base de données) · **Sévérité :** Moyenne · **Priorité :** Haute

### 🔹 Présentation

Une inversion globale des données a été identifiée : les matchs affichent les équipes dans des rôles opposés à la réalité officielle. Les équipes jouant à domicile (Home) sont étiquetées à l'extérieur (Away), et inversement.

### 🔹 Diagnostic technique

| 🔵 Source | Constat |
|-----------|---------|
| **Vérification cross-browser** | Le bug est reproduit sur Google Chrome, Microsoft Edge, Safari et les applications mobiles. Il est donc indépendant de la configuration du client. |
| **Analyse du payload** | L'inspection de la réponse brute de `fixtures/?event=5` montre une erreur de mapping à la source. Pour le match Chelsea vs Brentford (Chelsea reçoit officiellement à Stamford Bridge), les rôles sont inversés dans le JSON. |
| **Conclusion** | Le Front-end affiche fidèlement ce qu'il reçoit. La défaillance provient des pipelines d'ingestion de la base de données Back-end ou des requêtes d'extraction. |

Extrait du payload analysé :

```jsonc
{
  "code": 2645245,
  "event": 5,
  "finished": false,
  "team_h": "Brentford", // 🔵 Erreur : assigné comme équipe à domicile
  "team_a": "Chelsea"    // 🔵 Erreur : assigné comme équipe à l'extérieur
}
```

Valeurs attendues :

```jsonc
{
  "code": 2645245,
  "event": 5,
  "finished": false,
  "team_h": "Chelsea",   // 🔷 Correct : équipe à domicile (Stamford Bridge)
  "team_a": "Brentford"  // 🔷 Correct : équipe à l'extérieur
}
```

**Classification :** Sévérité **Moyenne** / Priorité **Haute**. Les données sportives sont corrompues de manière systémique et tous les utilisateurs sont impactés.

---

## 🔷 Compétences démontrées

| 🔹 Compétence | Description |
|---------------|-------------|
| **Isolation des causes racines** | Distinction entre plantages environnementaux côté client et anomalies de mapping de données côté serveur. |
| **Maîtrise des DevTools** | Exploitation des onglets Network (headers, payloads) et des traces de console asynchrones. |
| **Esprit orienté solution** | Fourniture de snippets de code correctifs et de recommandations en complément des rapports de bugs. |
| **Tests cross-browser** | Validation de la reproductibilité sur plusieurs navigateurs et sur mobile. |

---

## 🔷 Licence

Ce projet est open-source et disponible sous les termes de la [Licence MIT](LICENSE).
