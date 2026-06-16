# Projet Supply Chain - Détection d'inefficacités logistiques au Maroc

## Problème résolu

Les PME marocaines du secteur agro-alimentaire subissent un gaspillage estimé à 30% (données FAO) sans outil simple pour le détecter. Les données existent souvent dans des fichiers Excel non exploités, et les entreprises n'ont pas de Data Analyst dédié.

**Notre solution** permet de saisir facilement les données via un chatbot, calculer automatiquement les KPIs, détecter les anomalies et estimer l'impact financier.

## Liens

- **Botpress (Studio)** :  
  https://studio.botpress.cloud/30af47cb-1851-4bda-8e07-e602cd9796a0/flows/wf-main

- **n8n (Workflow)** :  
  https://hafoussa.app.n8n.cloud/workflow/evKcytFgOLT9TdrR

- **Google Sheets** :  
  https://docs.google.com/spreadsheets/d/1JgB3lLUh5seMxIYwovqpNLZhDJnM6Hx1_1_EOID6boo/edit?gid=0

- **GitHub** :  
  https://github.com/sarak122/khsara

## Guide d'utilisation

1. Ouvrir Botpress : [INSERER LIEN BOTPRESS]
2. Choisir "Saisie Donnees" dans le menu
3. Répondre aux 10 questions (produit, dates, quantités, pertes, délais)
4. Choisir "Dashboard KPIs" pour voir les indicateurs avec codes couleur
5. Choisir "Rapport d'anomalie" pour voir les anomalies et l'impact en MAD
6. Choisir "Comparaison sectorielle" pour comparer son taux avec la moyenne

## Architecture

- **Botpress** : Interface conversationnelle (chatbot) avec 4 flows
- **n8n** : Automatisation avec 4 workflows
- **Google Sheets** : Stockage des données et calcul des KPIs

### Flows Botpress

| Flow | Nom | Fonction |
|------|-----|----------|
| Flow 1 | Saisie Donnees | Formulaire de 10 questions, envoi à n8n |
| Flow 2 | Dashboard KPIs | Affichage des 5 KPIs avec codes couleur |
| Flow 3 | Rapport d'anomalie | Affichage des anomalies + impact MAD |
| Flow 4 | Comparaison sectorielle | Comparaison avec benchmarks |

### Workflows n8n

| Workflow | Nom | Fonction |
|----------|-----|----------|
| Workflow 1 | saisie_data | Webhook POST, écriture dans Google Sheets |
| Workflow 2 | Alerte_anomalie | Détection anomalies → LLM → email |
| Workflow 3 | rppt_hebdo | Cron (vendredi 17h) → rapport hebdomadaire |
| Workflow 4 | dash | Webhook GET, fournit les données à Botpress |

## KPIs calculés

| KPI | Formule | Seuil vert | Seuil orange | Seuil rouge |
|-----|---------|------------|--------------|-------------|
| Taux de perte | pertes / quantite_entrée × 100 | < 3% | 3-5% | > 5% |
| Taux de service | livraisons à temps / total | > 95% | 90-95% | < 90% |
| Rotation stock | quantite_sortie / stock_moyen | > 8 | 5-8 | < 5 |
| Écart délai | delai_reel - delai_prevu | < 1j | 1-2j | > 2j |

## Difficultés rencontrées

1. **Problème de connexion Botpress → n8n** : Résolu en utilisant les webhooks n8n en mode Production
2. **Format des dates** : Adaptation du format ISO pour Google Sheets
3. **Variables Botpress** : Utilisation de `workflow.xxx` au lieu de `session.xxx`
4. **Parsing CSV** : Gestion des virgules décimales (ex: 0,989 → 0.989)
5. **Affichage des rapports** : Nœud Text vide avec `return message`

## Code du projet

### Flow 1 - Saisie Donnees

```javascript
const produitValue = workflow.produit;
const motifValue = workflow.motif_perte;
const livraisonValue = workflow.live;
const dateEtree = workflow.dateEtree;
const dateSrt = workflow.dateSrt;
const qteEntree = parseInt(workflow.qte_entree) || 0;
const qteSortie = parseInt(workflow.qte_sortie) || 0;
const pertes = parseInt(workflow.perte) || 0;
const delaiPrevu = parseInt(workflow.delai_prevu) || 0;
const delaiReel = parseInt(workflow.delai_reel) || 0;

const payload = {
  timestamp: new Date().toISOString(),
  produit: produitValue,
  date_entree: dateEtree,
  date_sortie: dateSrt,
  quantite_entree: qteEntree,
  quantite_sortie: qteSortie,
  pertes_declarees: pertes,
  motif_perte: motifValue,
  delai_prevu: delaiPrevu,
  delai_reel: delaiReel,
  livraison_a_temps: livraisonValue
};

try {
  const response = await axios.post('https://hafoussa.app.n8n.cloud/webhook/saisie-dataa', payload);
  session.message = response.status === 200 ? "✅ Données enregistrées avec succès !" : "❌ Erreur: " + response.status;
} catch (error) {
  session.message = "❌ Erreur de connexion: " + error.message;
}
## Flow 2 - Dashboard KPIs
try {
  const response = await axios.get('https://hafoussa.app.n8n.cloud/webhook/dash');
  let rows = response.data;
  if (rows && !Array.isArray(rows)) rows = rows.data || rows.results || rows.items || [rows];
  
  const validRows = rows.filter(row => {
    const perte = parseFloat(row.taux_perte);
    const service = parseFloat(row.taux_service);
    return !isNaN(perte) && !isNaN(service);
  });
  
  if (validRows.length === 0) return "❌ Aucune donnée valide";
  
  let totalPerte = 0, totalService = 0, anomalies = 0;
  for (const row of validRows) {
    totalPerte += parseFloat(row.taux_perte);
    totalService += parseFloat(row.taux_service);
    if (row.flag_anomalie === 'CRITIQUE') anomalies++;
  }
  
  const avgPerte = (totalPerte / validRows.length).toFixed(1);
  const avgService = (totalService / validRows.length).toFixed(0);
  
  const getColor = (v, sv, so, inv = false) => {
    if (inv) return v >= sv ? "🟢" : v >= so ? "🟠" : "🔴";
    return v <= sv ? "🟢" : v <= so ? "🟠" : "🔴";
  };
  
  return `
📊 DASHBOARD KPIs 📊
${getColor(avgPerte, 3, 5)} Taux de perte moyen : ${avgPerte}%
${getColor(avgService, 95, 90, true)} Taux de service moyen : ${avgService}%
${anomalies > 2 ? "🔴" : anomalies > 0 ? "🟠" : "🟢"} Anomalies critiques : ${anomalies}
📈 Période : ${validRows.length} enregistrements`;
} catch (error) {
  return `❌ Erreur: ${error.message}`;
}
 ##Flow 3 - Rapport d'anomalie
try {
  const response = await axios.get('https://hafoussa.app.n8n.cloud/webhook/dash');
  const anomalies = response.data.filter(row => row.flag_anomalie === 'CRITIQUE' && row.produit);
  
  if (anomalies.length === 0) return "✅ Aucune anomalie critique à signaler.";
  
  let impactTotal = 0, details = "";
  for (const a of anomalies) {
    const perte = parseFloat(a.taux_perte);
    const perteExcedentaire = Math.max(0, perte - 5);
    const impact = perteExcedentaire * 1000;
    impactTotal += impact;
    details += `\n🔴 ${a.produit} : ${perte}% → ${impact} MAD`;
  }
  
  return `📋 RAPPORT D'ANOMALIE 📋\n🔍 ${anomalies.length} anomalie(s) :${details}\n💰 Impact total : ${impactTotal} MAD\n📌 Recommandations : Analyser les causes, renforcer les contrôles qualité, optimiser la chaîne logistique.`;
} catch (error) {
  return `❌ Erreur: ${error.message}`;
}
 ##Flow 4 - Comparaison sectorielle
try {
  let tauxPerte = parseFloat(workflow.taux || session.taux);
  if (isNaN(tauxPerte)) return "❌ Veuillez entrer un nombre valide (ex: 8)";
  
  const response = await axios.get('https://hafoussa.app.n8n.cloud/webhook/dash');
  let totalPerte = 0, count = 0;
  for (const row of response.data) {
    const perte = parseFloat(row.taux_perte);
    if (!isNaN(perte) && perte > 0 && perte <= 50) {
      totalPerte += perte;
      count++;
    }
  }
  
  if (count === 0) return "❌ Pas assez de données historiques";
  
  const moyenne = totalPerte / count;
  const alerte = moyenne * 1.5;
  const critique = moyenne * 2;
  
  let couleur, statut;
  if (tauxPerte <= moyenne) { couleur = "🟢"; statut = "excellent"; }
  else if (tauxPerte <= alerte) { couleur = "🟡"; statut = "acceptable"; }
  else if (tauxPerte <= critique) { couleur = "🟠"; statut = "préoccupant"; }
  else { couleur = "🔴"; statut = "critique"; }
  
  return `📊 COMPARAISON SECTORIELLE 📊\n${couleur} Votre taux : ${tauxPerte}%\n${couleur} Moyenne secteur : ${moyenne.toFixed(1)}%\nDiagnostic : Votre taux est ${statut}.\n${tauxPerte > moyenne ? `⚠️ Vous dépassez la moyenne de ${(tauxPerte - moyenne).toFixed(1)} points.` : "✅ Vous êtes en dessous de la moyenne."}`;
} catch (error) {
  return `❌ Erreur: ${error.message}`;
}
 ## Prompts utilisés:
You are a supply chain expert in Morocco.

Generate a WEEKLY PERFORMANCE REPORT as a PROFESSIONAL EMAIL.

Data for the week {{$json.semaine}}:
- Total entries: {{$json.nbLignes}}
- Anomalies detected: {{$json.nbAnomalies}}
- Average loss rate: {{$json.avgTauxPerte}}%
- Average service rate: {{$json.avgTauxService}}%

Use EXACTLY this format:

**Subject:** Rapport Hebdomadaire Supply Chain - Semaine du {{$json.semaine}}

**Bonjour l'équipe,**

**📊 Synthèse de la semaine :**
- Volume d'activité : X entrées
- Taux de perte moyen : X%
- Taux de service moyen : X%
- Anomalies critiques : X

**⚠️ Points d'attention :**
(2-3 lignes sur les problèmes identifiés)

**🎯 Actions prioritaires (3) :**
1. [Action 1 avec responsable suggéré]
2. [Action 2 avec responsable suggéré]
3. [Action 3 avec responsable suggéré]

**📈 Recommandations :**
(1-2 lignes)

**Cordialement,**
Système de Supervision Supply Chain

**---**
*Ce rapport est généré automatiquement par n8n*
## Résultats des tests

| Métrique | Résultat |
|----------|----------|
| Taux de détection | 100% (3/3 anomalies) |
| Faux positifs | 1 (≤ 2 acceptable) |
| Précision des KPIs | 100% |
| Délai de détection | < 1 minute |
| Qualité recommandations | 4/5 |

**Test dataset 30 jours** : 3 anomalies intentionnelles injectées (15/03, 22/03, 28/03) → toutes détectées ✅ → Impact financier total : 20 688 MAD

## ⚖️ Éthique et biais

- **Biais de seuil** : Seuils définis sans validation terrain → à valider avec experts
- **Biais de disponibilité** : Erreurs de saisie possibles → valider les données à l'entrée
- **Biais d'attribution** : Causes présentées comme "probables" → investigation humaine nécessaire
