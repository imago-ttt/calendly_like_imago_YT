# calendly_like_imago_YT

# Booking Widget — Calendly-like avec n8n

Widget de prise de rendez-vous 100% custom, sans dépendance à Calendly ou Fillout.  
Stack : **HTML/CSS/JS vanilla** côté frontend + **n8n** pour le backend + **Google Calendar**, **Airtable** et **Gmail**.

---

## Architecture

```
Frontend (index.html)
    │
    ├── POST /webhook/availability  →  Workflow 1 : retourne les créneaux libres
    └── POST /webhook/book          →  Workflow 2 : crée l'événement + envoie les emails
```

---

## Contenu du dossier

| Fichier | Rôle |
|---|---|
| `index.html` | Page de réservation (widget complet, autonome) |
| `workflow1-disponibilites.json` | Workflow n8n — créneaux disponibles |
| `workflow2-reservation.json` | Workflow n8n — confirmation de réservation |
| `README.md` | Ce fichier |

---

## Prérequis

- Une instance n8n (self-hosted ou cloud)  
- Un compte Google (Calendar + Gmail)  
- Un compte Airtable (optionnel — retirer le nœud si non utilisé)

---

## Installation — Étape par étape

### 1. Importer les workflows dans n8n

1. Ouvrir votre instance n8n
2. Cliquer sur **"Add workflow"** → **"Import from file"**
3. Importer `workflow1-disponibilites.json`
4. Répéter pour `workflow2-reservation.json`

---

### 2. Connecter Google Calendar

Dans n8n, créer une credential **Google Calendar OAuth2** :

1. Menu → **Credentials** → **Add credential** → choisir `Google Calendar OAuth2 API`
2. Suivre le flux OAuth (autoriser l'accès à votre compte Google)
3. **Dans les deux workflows**, sur chaque nœud Google Calendar :
   - Remplacer `REMPLACER_PAR_ID_CREDENTIAL_GCAL` par l'ID de votre credential
   - Remplacer `VOTRE_EMAIL_GCAL@gmail.com` par votre adresse Gmail

---

### 3. Connecter Gmail

Dans n8n, créer une credential **Gmail OAuth2** :

1. Menu → **Credentials** → **Add credential** → choisir `Gmail OAuth2 API`
2. Suivre le flux OAuth
3. **Dans le Workflow 2**, sur les deux nœuds Gmail :
   - Remplacer `REMPLACER_PAR_ID_CREDENTIAL_GMAIL` par l'ID de votre credential Gmail
   - Dans le nœud **"Notification interne"**, remplacer `VOTRE_EMAIL@gmail.com` par l'adresse qui recevra les notifications

---

### 4. Connecter Airtable (optionnel)

Si vous souhaitez sauvegarder les prospects dans Airtable :

1. Créer une base Airtable avec une table **Prospects** contenant les colonnes :  
   `Nom`, `Email`, `Téléphone`, `Motif`, `Message`, `Date RDV`, `Heure RDV`, `Statut`, `Source`
2. Dans n8n, créer une credential **Airtable Personal Access Token**
3. Dans le Workflow 2, nœud **"Airtable — Sauvegarder prospect"** :
   - Remplacer `REMPLACER_PAR_AIRTABLE_BASE_ID` par l'ID de votre base (visible dans l'URL Airtable)
   - Remplacer `REMPLACER_PAR_AIRTABLE_TABLE_ID` par l'ID de votre table
   - Remplacer `REMPLACER_PAR_ID_CREDENTIAL_AIRTABLE` par l'ID de votre credential

> **Si vous n'utilisez pas Airtable**, supprimez simplement le nœud dans le workflow et reliez directement "Créer événement GCal" → "Email confirmation — prospect".

---

### 5. Activer les workflows

Dans n8n, pour chaque workflow :
- Cliquer sur le toggle **"Active"** en haut à droite
- Copier l'URL du webhook affichée (ex: `https://votre-n8n.com/webhook/availability`)

---

### 6. Configurer le frontend

Ouvrir `index.html` et repérer le bloc de configuration en haut du script :

```javascript
var BOOKING_CONFIG = {
  availabilityUrl: 'https://VOTRE_INSTANCE_N8N/webhook/availability',
  bookUrl:         'https://VOTRE_INSTANCE_N8N/webhook/book',
  source:          'landing'
};
```

Remplacer les URLs par celles de vos webhooks n8n activés.

Personnaliser également :
- **Logo / nom** : balise `<span class="logo-name">` dans le header
- **Titre et description** : section `.info-panel` dans le HTML
- **Nom de l'organisateur** : `.organizer-name` et `.organizer-role`
- **Options du menu "Objet de l'appel"** : tableau `motifOptions` dans le script

---

### 7. Tester

Ouvrir `index.html` via un serveur local (ne pas ouvrir directement depuis `file://` — les requêtes fetch seront bloquées par CORS) :

```bash
# Python
python -m http.server 8080

# Node.js
npx serve .
```

Puis aller sur `http://localhost:8080` et effectuer une réservation complète.

---

## Fonctionnement des workflows

### Workflow 1 — Disponibilités

```
Webhook (POST /availability)
  └── Code : génère les créneaux de la journée (Lun–Ven 9h–19h30, Sam 10h–12h)
        └── Google Calendar : récupère tous les événements du jour
              └── Code : filtre les créneaux occupés → retourne { slots: [...] }
```

**Payload reçu :**
```json
{ "date": "2024-04-15" }
```

**Réponse :**
```json
{ "slots": ["09:00", "09:30", "10:00", ...] }
```

---

### Workflow 2 — Réservation

```
Webhook (POST /book)
  └── Code : calcule startISO / endISO avec gestion DST Europe/Paris
        └── Google Calendar : vérifie que le créneau est libre
              └── IF créneau pris ?
                    ├── OUI → Respond { success: false, error: "slot_taken" }
                    └── NON → Créer événement GCal
                                └── Airtable : enregistrer prospect
                                      └── Gmail : email de confirmation au prospect
                                            └── Gmail : notification interne
                                                  └── Respond { success: true }
```

**Payload reçu :**
```json
{
  "date": "2024-04-15",
  "time": "10:00",
  "nom": "Marie Martin",
  "email": "marie@exemple.fr",
  "telephone": "+33612345678",
  "motif": "decouverte",
  "message": "Je souhaite en savoir plus.",
  "source": "landing"
}
```

---

## Personnalisation des horaires

Les créneaux sont définis dans le nœud **"Générer les créneaux du jour"** du Workflow 1.  
Modifier les plages dans le code JavaScript du nœud :

```javascript
// Samedi : 10h–12h
for (let h = 10; h < 12; h++) { ... }

// Lundi–Vendredi : 9h–19h30
for (let h = 9; h < 20; h++) {
  const maxMin = (h === 19) ? 30 : 60;
  ...
}
```

La durée d'un créneau est de **30 minutes** (modifiable en changeant le `+= 30` et le calcul `+ 30 * 60 * 1000`).

---

## Questions fréquentes

**Les créneaux ne s'affichent pas ?**  
Vérifier que le workflow 1 est actif et que l'URL webhook dans `BOOKING_CONFIG` est correcte. Tester avec `curl` ou Postman :
```bash
curl -X POST https://votre-n8n.com/webhook/availability \
  -H "Content-Type: application/json" \
  -d '{"date":"2024-04-15"}'
```

**Le créneau est toujours marqué occupé ?**  
Vérifier que le nœud Google Calendar est bien configuré en mode `Event / Get All` (pas `Calendar / Get Availability`).

**L'email de confirmation n'arrive pas ?**  
S'assurer que la credential Gmail est correctement autorisée et que le nœud n'est pas en erreur dans les logs n8n.

