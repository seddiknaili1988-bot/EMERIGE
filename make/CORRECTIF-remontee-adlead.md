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
