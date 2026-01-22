# Script CSV User FreeRADIUS

## Description

Solution d'automatisation pour l'importation massive d'utilisateurs dans FreeRADIUS à partir d'un fichier CSV. Les mots de passe sont automatiquement chiffrés en MD5 et intégrés dans la configuration FreeRADIUS.

## Prérequis

- FreeRADIUS 3.0 ou supérieur
- Accès root au serveur
- Shell Bash

## Installation

### Étape 1 : Créer le répertoire de scripts

```bash
sudo mkdir -p /etc/script
cd /etc/script
```

### Étape 2 : Créer le script principal d'importation

```bash
sudo nano /etc/script/csvuser.sh
```

Copier le contenu suivant dans le fichier puis sauvegarder (Ctrl+O, Entrée, Ctrl+X) :

```bash
#!/bin/bash

# Script pour importer des utilisateurs depuis un CSV vers FreeRADIUS
# Format du CSV: username,password (une ligne par utilisateur)

CSV_FILE="$1"
SYNC_MODE="$2"
FREERADIUS_USERS="/etc/freeradius/3.0/users"

# Vérification que le fichier CSV est fourni
if [ -z "$CSV_FILE" ]; then
    echo "Usage: $0 <fichier.csv> [--sync]"
    echo "Format du CSV: username,password"
    echo ""
    echo "Options:"
    echo "  --sync : Supprime les utilisateurs qui ne sont plus dans le CSV"
    exit 1
fi

# Vérification que le fichier CSV existe
if [ ! -f "$CSV_FILE" ]; then
    echo "Erreur: Le fichier $CSV_FILE n'existe pas"
    exit 1
fi

# Vérification des permissions root
if [ "$EUID" -ne 0 ]; then
    echo "Erreur: Ce script doit être exécuté en tant que root"
    exit 1
fi

# Compteur d'utilisateurs
ADDED=0
SKIPPED=0
DELETED=0
TOTAL=0

echo "=== Traitement du fichier CSV ==="
echo "Fichier: $CSV_FILE"
if [ "$SYNC_MODE" = "--sync" ]; then
    echo "Mode: SYNCHRONISATION (suppression des utilisateurs absents du CSV)"
else
    echo "Mode: AJOUT UNIQUEMENT (utilisez --sync pour supprimer les absents)"
fi
echo ""

# Créer un fichier temporaire avec la liste des utilisateurs du CSV
TEMP_CSV_USERS=$(mktemp)
while IFS=',' read -r username password || [ -n "$username" ]; do
    [ -z "$username" ] && continue
    username="${username#"${username%%[![:space:]]*}"}"
    username="${username%"${username##*[![:space:]]}"}"
    echo "$username" >> "$TEMP_CSV_USERS"
done < "$CSV_FILE"

# Mode synchronisation : supprimer les utilisateurs qui ne sont plus dans le CSV
if [ "$SYNC_MODE" = "--sync" ]; then
    echo "=== Vérification des utilisateurs à supprimer ==="

    # Créer un fichier temporaire pour les utilisateurs à supprimer
    TEMP_TO_DELETE=$(mktemp)

    # Extraire tous les utilisateurs du fichier FreeRADIUS (ceux avec MD5-Password ou Cleartext-Password)
    grep -E "^[a-zA-Z0-9_.-]+ (MD5-Password|Cleartext-Password) :=" "$FREERADIUS_USERS" | awk '{print $1}' > "$TEMP_TO_DELETE.all"

    # Vérifier chaque utilisateur existant
    if [ -f "$TEMP_TO_DELETE.all" ] && [ -s "$TEMP_TO_DELETE.all" ]; then
        while read existing_user; do
            # Vérifier si l'utilisateur existe dans le CSV
            if ! grep -Fxq "${existing_user}" "$TEMP_CSV_USERS"; then
                echo "$existing_user" >> "$TEMP_TO_DELETE"
            fi
        done < "$TEMP_TO_DELETE.all"
    fi

    # Supprimer les utilisateurs identifiés
    if [ -f "$TEMP_TO_DELETE" ] && [ -s "$TEMP_TO_DELETE" ]; then
        while read user_to_delete; do
            echo "Suppression: $user_to_delete (absent du CSV)"

            # Échapper les caractères spéciaux pour sed
            escaped_user=$(printf '%s\n' "$user_to_delete" | sed 's/[[\.*^$/]/\\&/g')
            
            # Supprimer l'utilisateur et la ligne vide qui suit
            sed -i "/^${escaped_user} MD5-Password :=/d; /^${escaped_user} Cleartext-Password :=/d" "$FREERADIUS_USERS"

            DELETED=$((DELETED + 1))
        done < "$TEMP_TO_DELETE"
    else
        echo "✓ Aucun utilisateur à supprimer"
    fi

    # Nettoyer les fichiers temporaires
    rm -f "$TEMP_TO_DELETE" "$TEMP_TO_DELETE.all"

    echo ""
fi

# Nettoyer le fichier temporaire
rm -f "$TEMP_CSV_USERS"

# Lecture du CSV et traitement ligne par ligne
while IFS=',' read -r username password || [ -n "$username" ]; do
    # Ignorer les lignes vides
    [ -z "$username" ] && continue

    TOTAL=$((TOTAL + 1))

    # Supprimer les espaces en début et fin
    username="${username#"${username%%[![:space:]]*}"}"
    username="${username%"${username##*[![:space:]]}"}"

    password="${password#"${password%%[![:space:]]*}"}"
    password="${password%"${password##*[![:space:]]}"}"

    echo "[$TOTAL] Traitement: $username"

    # Vérifier si l'utilisateur existe déjà
    if grep -q "^$username " "$FREERADIUS_USERS"; then
        echo "Utilisateur déjà existant - ignoré"
        SKIPPED=$((SKIPPED + 1))
        echo ""
        continue
    fi

    echo "  → Génération du hash MD5..."
    # Générer le hash MD5 (format identique à echo -n "password" | md5sum)
    md5_hash=$(echo -n "$password" | md5sum | cut -d' ' -f1)
    echo "  → Hash: $md5_hash"

    echo "  → Ajout au fichier FreeRADIUS..."
    # Ajouter l'utilisateur au fichier FreeRADIUS (format simple sans classe)
    cat >> "$FREERADIUS_USERS" << EOF

$username MD5-Password := "$md5_hash"
EOF

    echo "  ✓ Utilisateur ajouté avec succès"
    ADDED=$((ADDED + 1))
    echo ""

done < "$CSV_FILE"

echo "=== Résumé ==="
echo "Total traité:             $TOTAL utilisateur(s)"
echo "Ajoutés:                  $ADDED utilisateur(s)"
echo "Ignorés (déjà existants): $SKIPPED utilisateur(s)"
if [ "$SYNC_MODE" = "--sync" ]; then
    echo "Supprimés:                $DELETED utilisateur(s)"
fi
echo ""
echo "=== Prochaine étape ==="
echo "Redémarrez FreeRADIUS pour appliquer les changements:"
echo "  systemctl restart freeradius"
```

Rendre le script exécutable :

```bash
sudo chmod +x /etc/script/csvuser.sh
```

### Étape 3 : Créer le fichier CSV des utilisateurs

```bash
sudo nano /etc/script/login_mdp_internet.csv
```

Format requis (username,password) :

```
user1,password123
user2,password456
user3,password789
```

Sauvegarder le fichier (Ctrl+O, Entrée, Ctrl+X).

### Étape 4 : Créer le script de synchronisation automatique

```bash
sudo nano /etc/script/sync_radius.sh
```

Copier le contenu suivant dans le fichier puis sauvegarder (Ctrl+O, Entrée, Ctrl+X) :

```bash
#!/bin/bash

systemctl stop freeradius.service
echo "╔════════════════════════════════════════╗"
echo "║      Le service Radius est Stoppé      ║"
echo "╚════════════════════════════════════════╝"
echo ""

# Compteur avant synchronisation
for i in 5 4 3 2 1; do
    echo -ne "Démarrage de la synchronisation du CSV dans : $i seconde(s)...\r"
    sleep 1
done

cd /etc/script
./csvuser.sh login_mdp_internet.csv --sync

# Compteur avant redémarrage
echo ""
for i in 5 4 3 2 1; do
    echo -ne "Redémarrage du service dans : $i seconde(s)...\r"
    sleep 1
done

systemctl restart freeradius.service
systemctl status freeradius.service
```

Rendre le script exécutable :

```bash
sudo chmod +x /etc/script/sync_radius.sh
```

## Avertissement critique

**ATTENTION : Le service FreeRADIUS DOIT être arrêté avant toute modification du fichier users.**

L'ajout d'utilisateurs pendant que le service est actif peut entraîner :
- Corruption du fichier de configuration
- Perte de données utilisateurs
- Défaillance du service d'authentification
- Incohérences dans la base d'utilisateurs
- Nécessité de restauration depuis une sauvegarde

**Toujours arrêter le service avant d'exécuter le script d'import.**

## Utilisation

### Méthode 1 : Synchronisation automatique (recommandée)

Cette méthode gère automatiquement l'arrêt et le redémarrage du service :

```bash
sudo /etc/script/sync_radius.sh
```

**Déroulement automatique :**
1. Arrêt du service FreeRADIUS
2. Pause de 5 secondes (compte à rebours visible)
3. Synchronisation des utilisateurs depuis le CSV
4. Pause de 5 secondes avant redémarrage
5. Redémarrage du service FreeRADIUS
6. Affichage du statut final

### Méthode 2 : Exécution manuelle

#### Mode ajout simple (sans --sync)

Ce mode ajoute uniquement les nouveaux utilisateurs présents dans le CSV.

**Comportement :**
- Les utilisateurs déjà existants dans FreeRADIUS sont ignorés
- Les utilisateurs absents du CSV mais présents dans FreeRADIUS sont conservés
- Aucune suppression n'est effectuée

```bash
sudo systemctl stop freeradius.service
sudo /etc/script/csvuser.sh /etc/script/login_mdp_internet.csv
sudo systemctl start freeradius.service
```

**Exemple de résultat :**

```
=== Résumé ===
Total traité:             15 utilisateur(s)
Ajoutés:                  10 utilisateur(s)
Ignorés (déjà existants): 5 utilisateur(s)
```

#### Mode synchronisation (avec --sync)

Ce mode synchronise complètement la base utilisateurs avec le contenu du CSV.

**Comportement :**
- Les nouveaux utilisateurs du CSV sont ajoutés
- Les utilisateurs déjà existants sont conservés
- Les utilisateurs absents du CSV mais présents dans FreeRADIUS sont SUPPRIMÉS

**ATTENTION : Cette opération est irréversible. Les utilisateurs supprimés ne peuvent être récupérés que depuis une sauvegarde.**

```bash
sudo systemctl stop freeradius.service
sudo /etc/script/csvuser.sh /etc/script/login_mdp_internet.csv --sync
sudo systemctl start freeradius.service
```

**Exemple de résultat :**

```
=== Vérification des utilisateurs à supprimer ===
Suppression: old_user1 (absent du CSV)
Suppression: old_user2 (absent du CSV)

=== Résumé ===
Total traité:             15 utilisateur(s)
Ajoutés:                  3 utilisateur(s)
Ignorés (déjà existants): 12 utilisateur(s)
Supprimés:                2 utilisateur(s)
```

## Différences entre les modes

| Caractéristique | Sans --sync | Avec --sync |
|----------------|-------------|-------------|
| Ajout de nouveaux utilisateurs | Oui | Oui |
| Conservation des existants | Oui | Oui |
| Suppression des absents du CSV | Non | Oui |
| Utilisation recommandée | Ajouts ponctuels | Synchronisation complète |
| Risque de perte de données | Aucun | Élevé si CSV incomplet |

## Sécurité

### Protection du fichier CSV

Le fichier CSV contient des mots de passe en clair. Protégez-le :

```bash
sudo chmod 600 /etc/script/login_mdp_internet.csv
sudo chown root:root /etc/script/login_mdp_internet.csv
```

### Sauvegarde avant synchronisation

Avant toute synchronisation en mode --sync, effectuez une sauvegarde :

```bash
sudo cp /etc/freeradius/3.0/users /etc/freeradius/3.0/users.backup.$(date +%Y%m%d_%H%M%S)
```

### Chiffrement MD5

Les mots de passe sont automatiquement chiffrés en MD5 avant insertion :

```
Format stocké : username MD5-Password := "5f4dcc3b5aa765d61d8327deb882cf99"
```

## Vérification et dépannage

### Vérifier le statut du service

```bash
sudo systemctl status freeradius.service
```

### Consulter les logs

```bash
sudo tail -f /var/log/freeradius/radius.log
```

### Tester l'authentification d'un utilisateur

```bash
sudo radtest username password localhost 0 testing123
```

### En cas de problème

1. Arrêter le service : `sudo systemctl stop freeradius.service`
2. Vérifier la syntaxe du fichier users : `sudo freeradius -X`
3. Restaurer depuis la sauvegarde si nécessaire
4. Redémarrer : `sudo systemctl start freeradius.service`

## Structure des fichiers

```
/etc/script/
├── csvuser.sh                    # Script d'import CSV
├── sync_radius.sh                # Script de synchronisation automatique
└── login_mdp_internet.csv        # Base utilisateurs

/etc/freeradius/3.0/
└── users                         # Fichier de configuration modifié
```

## Bonnes pratiques

1. Toujours arrêter le service avant modification
2. Effectuer des sauvegardes régulières du fichier users
3. Tester sur un environnement de développement avant production
4. Utiliser le mode --sync uniquement quand nécessaire
5. Vérifier le fichier CSV avant exécution
6. Consulter les logs après chaque modification
7. Maintenir un historique des sauvegardes

## Configuration pfSense pour l'authentification RADIUS

Une fois FreeRADIUS configuré avec vos utilisateurs, vous pouvez intégrer l'authentification RADIUS dans pfSense.

### Étape 1 : Ajouter le serveur RADIUS dans pfSense

Connectez-vous à l'interface web de pfSense et accédez au menu :

**System > User Manager > Authentication Servers**

Cliquez sur le bouton **Add** pour ajouter un nouveau serveur d'authentification.

<img width="1143" height="851" alt="image" src="https://github.com/user-attachments/assets/beb66308-2b22-499b-bb3d-9fc01e28dc60" />

### Étape 2 : Configuration du serveur RADIUS

Dans la zone **Server Settings**, effectuez la configuration suivante :

**Descriptive name:** RADIUS  
**Type:** RADIUS

![Configuration serveur RADIUS](https://d1ny9casiyy5u5.cloudfront.net/wp-content/uploads/2019/10/pfsense-freeradius.webp)

Dans la zone **RADIUS Server Settings**, effectuez la configuration suivante :

**Protocol:** PAP  
**Hostname or IP address:** [Adresse IP de votre serveur FreeRADIUS]  
**Shared Secret:** [Le secret partagé configuré dans clients.conf]  
**Services offered:** Authentication and Accounting  
**Authentication port:** 1812  
**Accounting port:** 1813  
**Authentication Timeout:** 5

![Paramètres du serveur RADIUS](https://d1ny9casiyy5u5.cloudfront.net/wp-content/uploads/2019/10/pfsense-radius-server-settings.webp)

**Important :** Remplacez l'adresse IP par celle de votre serveur FreeRADIUS et le secret partagé par celui que vous avez défini dans `/etc/freeradius/3.0/clients.conf`.

Cliquez sur **Save** pour enregistrer la configuration.

### Étape 3 : Tester l'authentification RADIUS

Accédez au menu **Diagnostics > Authentication**.

![Menu test authentification](https://d1ny9casiyy5u5.cloudfront.net/wp-content/uploads/2019/09/pfsense-diagnostics-authentication.webp)

Sélectionnez le serveur d'authentification **RADIUS**, entrez un nom d'utilisateur et son mot de passe configurés dans FreeRADIUS, puis cliquez sur **Test**.

![Test authentification FreeRADIUS](https://d1ny9casiyy5u5.cloudfront.net/wp-content/uploads/2019/10/Pfsense-Freeradius-authentication-test.webp)

Si le test réussit, vous devriez voir ce message :

![Test réussi](https://d1ny9casiyy5u5.cloudfront.net/wp-content/uploads/2019/09/pfsense-active-directory-login-test.webp)

### Étape 4 : Créer un groupe d'autorisation

Accédez au menu **System > User Manager**, puis à l'onglet **Groups** et cliquez sur **Add**.

![Gestionnaire de groupes](https://d1ny9casiyy5u5.cloudfront.net/wp-content/uploads/2019/09/pfsense-group-manager.webp)

Effectuez la configuration suivante :

**Group name:** pfsense-admin  
**Scope:** Remote  
**Description:** Groupe FreeRADIUS

![Création du groupe](https://d1ny9casiyy5u5.cloudfront.net/wp-content/uploads/2019/10/pfsense-freeradius-group.webp)

**Important :** Le nom du groupe **pfsense-admin** doit correspondre exactement à la valeur **Class** définie dans le fichier `/etc/freeradius/3.0/users` de FreeRADIUS.

Cliquez sur **Save**.

### Étape 5 : Attribuer les privilèges au groupe

Éditez le groupe **pfsense-admin** que vous venez de créer.

Dans la section **Assigned Privileges**, cliquez sur **Add** et sélectionnez :

**WebCfg - All pages**

![Permissions du groupe](https://d1ny9casiyy5u5.cloudfront.net/wp-content/uploads/2019/09/pfsense-active-directory-group-permission.webp)

Cette permission accorde un accès administrateur complet à l'interface web de pfSense.

Cliquez sur **Save** pour enregistrer.

### Étape 6 : Activer l'authentification RADIUS

Accédez au menu **System > User Manager**, puis à l'onglet **Settings**.

![Menu paramètres d'authentification](https://d1ny9casiyy5u5.cloudfront.net/wp-content/uploads/2019/09/pfsense-authentication-settings-menu.webp)

Dans **Authentication Server**, sélectionnez votre serveur **RADIUS**.

![Activation RADIUS](https://d1ny9casiyy5u5.cloudfront.net/wp-content/uploads/2019/10/pfsense-enable-radius-authentication-freeradius.webp)

Cliquez sur **Save & Test**.

### Étape 7 : Test de connexion

Déconnectez-vous de l'interface pfSense.

Tentez une nouvelle connexion en utilisant les identifiants configurés dans FreeRADIUS :

**Username:** admin  
**Password:** [Le mot de passe configuré dans FreeRADIUS]

![Connexion pfSense](https://d1ny9casiyy5u5.cloudfront.net/wp-content/uploads/2019/09/Pfsense-login.webp)

Si la connexion réussit, votre authentification RADIUS fonctionne correctement.

### Points de vérification

**En cas de problème de connexion, vérifiez :**

1. Le service FreeRADIUS est démarré : `systemctl status freeradius.service`
2. L'adresse IP dans clients.conf correspond à l'IP de pfSense
3. Le secret partagé est identique dans clients.conf et dans pfSense
4. Le nom du groupe dans pfSense correspond exactement à la valeur Class dans users
5. Les logs FreeRADIUS : `tail -f /var/log/freeradius/radius.log`
6. Le port 1812 est accessible depuis pfSense vers le serveur FreeRADIUS
