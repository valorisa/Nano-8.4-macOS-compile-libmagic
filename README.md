# Compilation de Nano 8.4 sur macOS (Intel) avec libmagic et UTF-8

Ce document détaille les étapes suivies pour compiler l'éditeur de texte GNU Nano version 8.4 depuis ses sources sur un système macOS avec une architecture Intel (x86_64). L'objectif principal était d'activer spécifiquement le support pour `libmagic` (détection du type de fichier) et `UTF-8` (encodage de caractères étendu).

## Objectif Initial

Compiler et installer nano 8.4 avec les options `--enable-libmagic` et `--enable-utf8`. La version initialement installée (probablement via Homebrew ou une compilation précédente) ne possédait pas ces options :

```bash
darkfairie@iMacdeIsabelle2-001 macOS-Nano-Compile % nano --version
GNU nano, version 8.4
(C) 2025 la Free Software Foundation et d'autres contributeurs
Compilé avec les options : --disable-libmagic --disable-utf8 
```

## Environnement

*   **Système d'exploitation :** macOS (testé sur une version compatible Intel, ex: Sonoma `darwin24.4.0`)
*   **Architecture :** Intel (x86_64)
*   **Gestionnaire de paquets :** Homebrew

## Prérequis

Avant de commencer, assurez-vous que les outils suivants sont installés :

1.  **Xcode Command Line Tools :** Fournit le compilateur C (clang), `make`, et d'autres outils de développement essentiels.
    ```bash
    xcode-select --install 
    ```

2.  **Homebrew :** Utilisé pour installer facilement les bibliothèques dépendantes. (Instructions sur [brew.sh](https://brew.sh/))
    ```bash
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    ```

3.  **Bibliothèques Dépendantes (via Homebrew) :**
    *   **`file` (pour `libmagic`) :** Nécessaire pour l'option `--enable-libmagic`.
        ```bash
        brew install file
        ```
    *   **`ncurses` (pour `ncursesw`) :** Requis pour le support correct de l'UTF-8 (`--enable-utf8`) dans le terminal. La version système de macOS est souvent insuffisante.
        ```bash
        brew install ncurses 
        ```
    *   **`gettext` :** Souvent nécessaire pour le support NLS (National Language Support / traductions). Peut être installé comme dépendance d'autres paquets ou explicitement.
        ```bash
        # Généralement installé comme dépendance, mais au besoin :
        # brew install gettext
        ```
    *   **`texinfo` (Optionnel, mais résout un problème de build) :** Nécessaire pour générer la documentation au format `.info` et `.html` lors de l'étape `make`.
        ```bash
        brew install texinfo
        ```
    *   **`groff` (Optionnel, mais résout un avertissement) :** Nécessaire pour générer la documentation HTML lors de la configuration et éviter un avertissement.
        ```bash
        brew install groff
        ```

## Étapes de Compilation (incluant les erreurs et corrections)

1.  **Créer un Répertoire de Travail**
    ```bash
    mkdir macOS-Nano-Compile
    cd macOS-Nano-Compile
    ```

2.  **Télécharger les Sources de Nano 8.4**
    *   Première tentative (échec silencieux) :
        ```bash
        # Échec initial, 0 octet téléchargé
        curl -O https://ftpmirror.gnu.org/nano/nano-8.4.tar.xz 
        ```
    *   Correction : Utiliser `-L` pour suivre les redirections, `-v` pour le mode verbeux et un miroir différent si nécessaire.
        ```bash
        # Téléchargement réussi (exemple avec options et miroir principal)
        curl -LvO https://ftp.gnu.org/gnu/nano/nano-8.4.tar.xz 
        # Ou depuis nano-editor.org:
        # curl -LvO https://www.nano-editor.org/dist/v8/nano-8.4.tar.xz
        ```

3.  **Extraire les Sources**
    ```bash
    tar -xf nano-8.4.tar.xz
    ```

4.  **Naviguer dans le Dossier des Sources**
    ```bash
    cd nano-8.4
    ```

5.  **Première Tentative de Configuration**
    *   Commande initiale :
        ```bash
        # Tentative de configuration simple
        ./configure --prefix=/usr/local --enable-libmagic --enable-utf8 PKG_CONFIG_PATH="$(brew --prefix)/lib/pkgconfig"
        ```
    *   Erreur rencontrée : Support UTF-8 insuffisant détecté.
        ```
        checking whether to enable UTF-8 support... yes
        [...]
        checking for ncursesw... no
        [...]
        checking for initscr in -lncurses... yes
        configure: error: 
          *** UTF-8 support was requested, but insufficient support was
          *** detected in your curses and/or C libraries. [...]
        ```
    *   Cause : Le script `configure` n'a pas trouvé ou utilisé la version `ncursesw` (avec support wide-char) nécessaire pour l'UTF-8.

6.  **Correction : Utiliser `ncurses` de Homebrew Explicitment**
    *   Vérifier/Installer `ncurses` (déjà fait dans les prérequis, mais pour confirmer) :
        ```bash
        brew install ncurses 
        # (Homebrew indiquera s'il est déjà installé)
        ```
    *   Relancer `configure` en spécifiant les chemins d'inclusion (`CPPFLAGS`) et de liaison (`LDFLAGS`) pour la version `ncurses` de Homebrew. Le `PKG_CONFIG_PATH` aide aussi.
        ```bash
        # Nettoyage avant re-configuration (si un Makefile existait déjà)
        # make clean  
        # Configuration corrigée
        ./configure --prefix=/usr/local \
                    --enable-libmagic \
                    --enable-utf8 \
                    PKG_CONFIG_PATH="$(brew --prefix)/lib/pkgconfig" \
                    CPPFLAGS="-I$(brew --prefix)/opt/ncurses/include" \
                    LDFLAGS="-L$(brew --prefix)/opt/ncurses/lib" 
        ```
    *   Résultat : La configuration réussit cette fois, détectant `ncursesw` et `libmagic`.
        ```
        [...]
        checking for wget_wch in -lncursesw... yes
          The curses library to be used is: ncursesw
        [...]
        checking for magic.h... yes
        checking for magic_open in -lmagic... yes
        [...]
        configure: creating ./config.status
        [...]
        ```

7.  **Vérification de la Configuration (`libmagic` non défini)**
    *   Malgré la détection de `libmagic` par `configure`, une vérification du fichier `config.h` généré révèle que le symbole nécessaire n'est pas défini :
        ```bash
        grep ENABLE_LIBMAGIC config.h
        # (Aucune sortie !)
        ```
    *   Cause : Probablement un bug ou une condition non remplie dans le script `configure` de nano qui empêche `#define ENABLE_LIBMAGIC 1` d'être écrit dans `config.h`, même si les tests de présence de la bibliothèque réussissent. Sans ce define, le code utilisant `libmagic` ne sera pas compilé.

8.  **Correction : Forcer la Définition `ENABLE_LIBMAGIC` lors de la Compilation**
    *   Identifier les `CFLAGS` par défaut utilisés par `configure` :
        ```bash
        grep '^CFLAGS\s*=' Makefile
        # Exemple de sortie : CFLAGS = -g -O2 -Wall
        ```
    *   Lancer `make` en ajoutant `-DENABLE_LIBMAGIC=1` aux `CFLAGS` récupérés :
        ```bash
        make CFLAGS="-DENABLE_LIBMAGIC=1 -g -O2 -Wall" 
        ```

9.  **Erreur de Compilation (`makeinfo` manquant)**
    *   La commande `make` précédente échoue lors de la génération de la documentation :
        ```
        [...] makeinfo: command not found
        WARNING: 'makeinfo' is missing on your system. [...]
        make[2]: *** [nano.html] Error 1
        [...]
        ```
    *   Cause : L'outil `makeinfo` (du paquet `texinfo`) est nécessaire pour créer la documentation HTML/Info, et il n'est pas installé.

10. **Correction : Installer `texinfo`**
    *   Installer `texinfo` via Homebrew (déjà fait dans les prérequis mis à jour).
        ```bash
        brew install texinfo
        ```
    *   Relancer la compilation avec les `CFLAGS` forcés :
        ```bash
        make CFLAGS="-DENABLE_LIBMAGIC=1 -g -O2 -Wall" 
        ```
    *   Résultat : La compilation se termine avec succès (des avertissements `ranlib: ... has no symbols` peuvent apparaître mais sont sans danger).

11. **Installation**
    *   Installer nano dans `/usr/local` (nécessite les droits administrateur) :
        ```bash
        sudo make install
        ```

## Vérification Finale

Après l'installation, fermer et rouvrir le terminal (ou recharger la configuration du shell, ex: `source ~/.zshrc`) pour s'assurer que le nouveau binaire est utilisé.

1.  **Vérifier le chemin de l'exécutable :**
    ```bash
    which nano
    # Doit afficher : /usr/local/bin/nano
    ```

2.  **Vérifier la version et les options de compilation :**
    ```bash
    nano --version
    ```
    La sortie devrait maintenant inclure les options activées :
    ```
     GNU nano, version 8.4
     (C) 2025 la Free Software Foundation et d'autres contributeurs
     Compilé avec les options : --enable-libmagic --enable-utf8 
    ```
    *(Note : L'ordre exact des options peut varier)*

## Résumé des Problèmes Rencontrés et Solutions

*   **Problème :** Échec du téléchargement initial avec `curl`.
    **Solution :** Utiliser `curl -LvO` et essayer différents miroirs.
*   **Problème :** Erreur `configure` liée au support UTF-8 (`ncursesw` non trouvé/utilisé).
    **Solution :** Installer `ncurses` via Homebrew et utiliser `CPPFLAGS` et `LDFLAGS` dans `configure` pour pointer explicitement vers les chemins de `ncurses` de Homebrew.
*   **Problème :** `libmagic` détecté par `configure` mais non activé dans la compilation finale (symbole `ENABLE_LIBMAGIC` manquant dans `config.h`).
    **Solution :** Forcer la définition du symbole lors de la compilation en passant `CFLAGS="-DENABLE_LIBMAGIC=1 ..."` à la commande `make`.
*   **Problème :** Échec de `make` car `makeinfo` est manquant (pour la documentation).
    **Solution :** Installer `texinfo` via Homebrew (`brew install texinfo`).

## Notes Finales

*   Il est conseillé de conserver le dossier des sources (`nano-8.4`) si vous souhaitez ultérieurement désinstaller cette version compilée via `sudo make uninstall` (exécuté depuis ce même dossier).
*   Les commandes `make clean` (supprime les fichiers objets) et `make distclean` (supprime les fichiers objets et les fichiers générés par `configure`) peuvent être utiles pour nettoyer le répertoire avant de retenter une configuration/compilation.
