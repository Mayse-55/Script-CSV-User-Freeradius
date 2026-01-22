# Script-CSV-User-Freeradius
Script qui permet d'ajouter les utilisateurs automatiquement dans le fichier client via csv 

Chiffrement en MD5 ( Penssée a éteindre le service avant )

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
    grep -E "^[a-zA-Z0-9_-]+ (MD5-Password|Cleartext-Password) :=" "$FREERADIUS_USERS" | awk '{print $1}' > "$TEMP_TO_DELETE.all"

    # Vérifier chaque utilisateur existant
    if [ -f "$TEMP_TO_DELETE.all" ]; then
        while read existing_user; do
            # Vérifier si l'utilisateur existe dans le CSV
            if ! grep -q "^${existing_user}$" "$TEMP_CSV_USERS"; then
                echo "$existing_user" >> "$TEMP_TO_DELETE"
            fi
        done < "$TEMP_TO_DELETE.all"
    fi

    # Supprimer les utilisateurs identifiés
    if [ -f "$TEMP_TO_DELETE" ] && [ -s "$TEMP_TO_DELETE" ]; then
        while read user_to_delete; do
            echo "Suppression: $user_to_delete (absent du CSV)"

            # Supprimer l'utilisateur et ses attributs (jusqu'à la ligne vide suivante)
            sed -i "/^${user_to_delete} /,/^$/d" "$FREERADIUS_USERS"

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

