# Application Web Fantasy Football - Rapport d'Investigation QA (Bugs)

Ce dépôt constitue un projet de portfolio QA professionnel, détaillant l'analyse des causes racines (Root-Cause Analysis), les diagnostics techniques et les correctifs suggérés pour deux bugs critiques découverts sur une application web de Fantasy Football.

Ces investigations démontrent une maîtrise avancée des outils **DevTools du navigateur (logs Console/Network)**, de l'analyse des **payloads d'API**, de l'évaluation **cross-browser**, ainsi que des concepts de **robustesse et de programmation défensive**.

---

## 🛑 Bug 1 : Gel de l'UI lors de la navigation entre les Journées (Côté Client / Problème de Robustesse)

### 📋 Présentation
Lors de la navigation entre les journées (Gameweeks) à l'aide des boutons fléchés `<` et `>`, l'interface utilisateur se fige complètement et ne répond plus. Pourtant, la communication réseau en arrière-plan avec le serveur s'exécute parfaitement.

### 🔍 Diagnostic Technique (Analyse de la cause racine)
* **Analyse de l'onglet Network :** Le clic sur le bouton de navigation déclenche avec succès une requête API vers `/api/fixtures/?event=X`. Le serveur répond correctement avec un statut **`HTTP 200 OK`** et renvoie le payload JSON attendu.
* **Analyse de l'onglet Console :** La console enregistre une exception non gérée : `Failed to load resource: the server responded with a status of 403 ()` ciblant des scripts externes publicitaires et de suivi (`googletag`/`PubAdsService`).
* **L'anomalie :** La logique de rendu du Front-end est étroitement couplée à l'exécution de ces scripts tiers. Lorsqu'un environnement client (comme un bloqueur de publicités agressif ou un outil de traduction automatique du navigateur) bloque le script externe, le thread d'exécution JavaScript principal crashe **avant** d'atteindre la séquence de rafraîchissement de l'UI (DOM re-render).
* **Classification :** **Sévérité Haute / Priorité Basse** (Bloque une fonctionnalité clé de l'utilisateur mais nécessite un déclencheur environnemental spécifique côté client).

### 🛠️ Correctif Code Suggéré (Programmation Défensive)
Le cycle de vie du rendu de l'interface doit être découplé des scripts externes non critiques. Envelopper l'injection de la dépendance externe dans un bloc `try...catch` isolé garantit que même si les scripts de tracking échouent, le DOM se met à jour normalement.

```javascript
// Correctif proposé par la QA
async function handleGameweekNavigation(nextEventId) {
    try {
        // 1. Récupération sécurisée des données principales
        const response = await fetch(`/api/fixtures/?event=${nextEventId}`);
        if (!response.ok) throw new Error('Échec de la requête API');
        const data = await response.json();

        // 2. Rendu immédiat de l'UI pour garantir une expérience utilisateur fluide
        updateGameweekDOM(data); 

        // 3. Isolation sécurisée de l'exécution du script tiers
        try {
            await loadExternalTrackingScript();
        } catch (resourceError) {
            console.warn("Script externe bloqué (403), rendu de l'UI non affecté.", resourceError);
        }

    } catch (criticalError) {
        console.error("Erreur critique dans le thread de navigation de l'UI :", criticalError);
        showFriendlyErrorToUser(); 
    }
}
```

---

## 🚨 Bug 2 : Inversion des Paramètres de Lieu des Matchs (API Back-end & Mapping Base de Données)

### 📋 Présentation
Une anomalie globale d'inversion des données a été identifiée : les matchs affichent les équipes dans des rôles opposés à la réalité officielle. Les équipes jouant à domicile (Home) sont étiquetées à l'extérieur (Away), et inversement.

### 🔍 Diagnostic Technique (Analyse de la cause racine)
* **Vérification Cross-Browser :** Ce bug persiste systématiquement sur tous les principaux moteurs de rendu (Google Chrome, Microsoft Edge, Safari) ainsi que sur les applications mobiles, ce qui prouve qu'il est totalement indépendant de la configuration du client.
* **Analyse approfondie du Payload :** L'inspection du flux de données API brut de `fixtures/?event=5` révèle une corruption structurelle à la source. Pour le match Chelsea vs Brentford (où Chelsea accueille officiellement à Stamford Bridge), le serveur transmet des paramètres de mapping JSON erronés :

```json
{ 
  "code": 2645245, 
  "event": 5, 
  "finished": false, 
  "team_h": "Brentford", // ❌ ERREUR : Assigné comme équipe à domicile
  "team_a": "Chelsea"    // ❌ ERREUR : Assigné comme équipe à l'extérieur
}
```
* **Conclusion :** La couche Front-end traite et affiche fidèlement la logique structurée qu'elle reçoit ; la faille provient entièrement des pipelines d'ingestion de la base de données Back-end ou des requêtes d'extraction.
* **Classification :** **Sévérité Moyenne / Priorité Haute** (Corrompt les données sportives de manière systémique sur toute la plateforme, impactant tous les utilisateurs simultanément).

---

## 🧪 Compétences Démontrées
* **Isolation des causes racines :** Différenciation claire entre les plantages environnementaux côté client (Front-end) et les anomalies de mapping de données côté serveur (Back-end).
* **Maîtrise des DevTools :** Utilisation experte des onglets Network Headers, Payloads et des traces de consoles asynchrones.
* **Esprit d'analyse orienté solution :** Fourniture de snippets de code correctifs et de recommandations de développement en complément des rapports de bugs.

---
## 📄 Licence
Ce projet est open-source et disponible sous les termes de la [Licence MIT](LICENSE).
