fichier serveur:

FIFO=./fif
PW_FILE=./PW
KEY_FILE=./KEY

PASSWORD=$(<"$PW_FILE")
KEY=$(<"$KEY_FILE")

[ -p "$FIFO" ] && rm "$FIFO"
mkfifo "$FIFO"

# Chiffrement monoalphabétique
mono_encrypt() {
    echo "$1" | tr 'A-Za-z' "$KEY"
}

# Déchiffrement (clé inversée)
mono_decrypt() {
    local ALPHABET="ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
    echo "$1" | tr "$KEY" "$ALPHABET"
}

interpret() {
    echo "Bienvenue sur le serveur"
    echo "Veuillez entrer le mot de passe :"
    read -r entered_password
    if [[ "$entered_password" != "$PASSWORD" ]]; then 
        echo "Mot de passe incorrect. Connexion refusée."
        exit 1
    fi

    echo "Mot de passe accepté. Connexion établie à $(date)"

    while read -r encrypted_line; do
        decrypted_line=$(mono_decrypt "$encrypted_line")
        echo "Commande reçue (chiffrée) : $encrypted_line" >&2
        echo "Commande déchiffrée : $decrypted_line" >&2

        if [[ "$decrypted_line" == "exit" ]]; then 
            response="Merci pour votre connexion"
            encrypted_response=$(mono_encrypt "$response")
            echo "$encrypted_response"
            break
        else 
            result=$(eval "$decrypted_line" 2>&1)
            encrypted_result=$(mono_encrypt "$result")
            echo "$encrypted_result"
        fi
    done
}

trap "rm -f $FIFO; echo 'Serveur arrêté.'; exit" INT

echo "Démarrage du serveur..."
echo "Serveur actif, en attente de connexions sur le port 12355..."

while true; do
    nc -l localhost 12355 <"$FIFO" | interpret >"$FIFO"
    echo "Client déconnecté, en attente d'une nouvelle connexion..."
done

fichier client:
#!/bin/bash

echo "Connexion au serveur..."

read -p "Mot de passe : " PASSWORD
read -p "Clé de chiffrement (52 lettres - A-Za-z dans l’ordre que tu veux) : " KEY

# Chiffrement
mono_encrypt() {
    echo "$1" | tr 'A-Za-z' "$KEY"
}

# Déchiffrement
mono_decrypt() {
    local ALPHABET="ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
    echo "$1" | tr "$KEY" "$ALPHABET"
}

while true; do
    read -p "Commande à envoyer (exit pour quitter) : " CMD

    # Envoi initial du mot de passe une seule fois
    if [[ -z "$SENT_PASSWORD" ]]; then
        echo "$PASSWORD"
        SENT_PASSWORD=true
    fi

    ENCRYPTED_CMD=$(mono_encrypt "$CMD")
    
    RESPONSE=$( (echo "$ENCRYPTED_CMD"; sleep 1) | nc localhost 12355)

    echo "Réponse chiffrée : $RESPONSE"
    echo -n "Réponse déchiffrée : "
    mono_decrypt "$RESPONSE"

    if [[ "$CMD" == "exit" ]]; then
        break
    fi
done
