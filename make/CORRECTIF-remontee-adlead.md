# Correctif remontée leads Adlead – Emerige Puteaux (ID 716)

Anomalies signalées par Ivan Chiarami le 07/10/2026 et corrections à faire dans Make.

## 1. Doublons (lead n°1 transmis 3 fois)

Cause probable : le module HTTP Adlead est exécuté plusieurs fois pour le même lead. Trois pistes à vérifier dans l'historique du scénario :
- le déclencheur est déclenché plusieurs fois (par exemple « Watch records » Airtable qui se redéclenche à chaque mise à jour du lead, ou un webhook appelé plusieurs fois) ;
- des exécutions incomplètes ou des erreurs ont été relancées automatiquement (option « Allow storing incomplete executions » ou une directive Retry/Break) ;
- deux scénarios actifs envoient vers Adlead (un pour Meta et un pour la landing, ou le scénario de test resté actif).

Correction :
1. Ajouter un **Data store « emerige_adlead_envoyes »** dont la clé est l'identifiant du lead : `leadgen_id` pour Meta, ou un ID unique de soumission pour la landing (à défaut, email + téléphone).
2. Avant le module HTTP : **Data store > Get a record** sur cette clé, puis un filtre « le record n'existe pas ».
3. Après un HTTP 200/201 : **Data store > Add a record** avec la clé, la date et l'ID du lead Adlead renvoyé.
4. Si le déclencheur est Airtable : passer en « Watch records » sur le champ **Created time**, pas sur Last modified.
5. Désactiver le scénario de test s'il est encore actif.

## 2. Canal d'origine (faux sur 100 % des leads)

Doc Adlead : https://docs.adlead.immo/v1/leads.html#canal-d-origine
Il ne faut que deux valeurs, choisies dynamiquement selon la provenance du lead :

| Provenance | key | name |
|---|---|---|
| Formulaire Meta (Lead Ads) | `lead-ads` | `Lead Ads` |
| Landing page | `landing` | `Landing` |

Dans le corps JSON du module HTTP :
```
"<champ canal d'origine>": {
  "key": "{{if(<provenance> = \"meta\"; \"lead-ads\"; \"landing\")}}",
  "name": "{{if(<provenance> = \"meta\"; \"Lead Ads\"; \"Landing\")}}"
}
```
Le nom exact du champ est à reprendre dans la section « canal d'origine » de la doc Adlead. Si le flux Meta et le flux landing passent par deux branches de router, on peut aussi mettre les valeurs en dur dans chaque branche.

## 3. Medium et UTM Campaign (incohérents : 1 sur 3 juste pour le lead n°1, faux pour le lead n°2)

Cause probable : `tracking_medium` et `tracking_campaign` sont mappés depuis des variables qui sont vides ou qui changent d'une exécution à l'autre, par exemple des UTM d'URL inexistants pour un lead Meta natif, ou le nom de campagne/adset Meta. La source, elle, est juste : elle est probablement fixée en dur.

Correction :
- **Meta Lead Ads** : il n'y a pas d'UTM dans un formulaire natif. Fixer les valeurs en dur selon la nomenclature UTM d'Emerige :
  - `tracking_source` : inchangé (déjà correct)
  - `tracking_medium` : valeur attendue par Emerige, à confirmer (ex. `paid-social`)
  - `tracking_campaign` : valeur attendue par Emerige, à confirmer (ex. `puteaux-en-vue`)
- **Landing page** : mapper `utm_medium` et `utm_campaign` depuis les champs cachés du formulaire. Si ces champs sont vides, reprendre par défaut les mêmes valeurs fixes (`ifempty(utm_medium; "...")`).

Demander à Ivan les valeurs exactes de medium et de campaign attendues dans Adlead.

## 4. Recette après correction
1. Envoyer un lead test Meta (outil de test Lead Ads) et un lead test landing.
2. Vérifier : un seul lead par soumission, canal `lead-ads` ou `landing`, et source, medium et campaign conformes.
3. Faire supprimer les 2 tests et les 2 doublons du lead n°1 par Emerige.

## 5. Incident du 08/10/2026 : « Enter valid JSON in the request body »

Exécution `c200a67bf65e4235b58a6f9e68744348` (scénario 9903104), erreur du module HTTP :
`Bad control character in string literal in JSON at position 414 (line 19 column 37)`.

Cause : le corps était saisi en mode **JSON string**, et les valeurs du formulaire y étaient collées telles quelles. La ligne 19 correspond à `"message": "{{3.data.avez-vous_une_attente_particulière_?}}"`. Le prospect a tapé un retour à la ligne dans sa réponse libre, ce qui rend le JSON invalide. Un guillemet `"` ou un antislash `\` dans n'importe quel champ (nom, message…) aurait produit la même erreur. Make a ensuite désactivé le scénario.

Correction appliquée dans Make :
- Module HTTP (id 35) : **Body input method = Data structure**, avec la structure « Adlead - corps lead Emerige (POST /leads) » (id 639404). Make échappe désormais lui-même les caractères réservés du JSON.
- Mapping inchangé : `tracking_origin` = `{"key":"lead-ads","name":"Lead Ads"}`, `tracking_source` = `facebook`, `tracking_medium` = `form`, `tracking_campaign` = `{{3.campaignName}}`.
- `property_rooms` reçoit désormais le tableau Meta tel quel (une entrée par typologie cochée), au lieu d'une seule chaîne qui les concaténait.
- Les modules orphelins de l'ancienne version (router 19 et ses modules) ont été retirés du canevas. Le blueprint d'avant correctif est conservé dans `REMONTEE_DES_LEADS_META_ADS_TEASING_OCT-2026_AVANT-CORRECTIF-JSON.blueprint.json`.
- Module Airtable « Create a record » (id 29) : retrait des deux collections vides « Responsable (assigné) » et « Assigné à », qui partaient en `{}` (erreur 422 `Cannot parse value "{}"`).
- Scénario réactivé. Le lead en échec (Rita, ID Lead Meta `28446973588256604`) est bien arrivé dans Adlead, et l'enregistrement Airtable a été créé par un replay pendant lequel le module HTTP était temporairement filtré, pour ne pas le renvoyer une nouvelle fois.

⚠️ Doublon probable dans Adlead : à la réactivation de 07:08, Make a traité en même temps le webhook resté en file d'attente (exécution `13f8f412…`) et le replay manuel (`445b18d5…`). Les deux exécutions ont passé le module HTTP. Il faut demander à Emerige / Adlead de vérifier et de supprimer le doublon de ce lead du 08/10 (Rita).

Règle à retenir : dans un module HTTP Make, ne jamais injecter un champ saisi par un utilisateur dans un corps en « JSON string ». Utiliser une Data structure, ou à défaut le module JSON > Create JSON.

## 6. Retour d'Ivan du 09/10/2026 : canal d'origine « Autre » et campagne fausse

D'après la doc Adlead (section « Noeud lead ») :
- `tracking_origin` est un **texte** qui contient seulement la clé : `"tracking_origin": "lead-ads"` (et `"landing"` pour la landing page). L'objet `{"key":"lead-ads","name":"Lead Ads"}` n'est pas reconnu, d'où « Autre ».
- `tracking_campaign` doit valoir `0926-avp-all-n-puteaux2-teasing-13791` (valeur donnée par Ivan le 07/10). La campagne Meta n'a jamais été renommée, donc `{{3.campaignName}}` envoyait `PUTEAUX.EMERIGE.TEASING.1026.FORM`.

Corrections à faire dans Make :
1. Structure de données « Adlead - corps lead Emerige (POST /leads) » (id 639404) : passer `lead > tracking_origin` du type Collection au type Texte.
2. Module HTTP : `tracking_origin` = `lead-ads` et `tracking_campaign` = `0926-avp-all-n-puteaux2-teasing-13791`.

À vérifier aussi dans la doc : `contact.title` attend `mr` ou `ms` (on envoie `m`), et `property_rooms` attend des clés du type `T2` ou `T3` (Meta envoie `2_pièces`).
