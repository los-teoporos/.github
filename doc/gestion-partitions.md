# Gestion des partitions

## Besoin

- Générer les PDFs et les Audios à partir des Musescores

## Contraintes

- Impossible d'installer le logiciel Musescore sur la plupart des hébergement web, qui permet de lire les Musescores et de générer les PDFs et les Audios
- Versionner les Musescores

## Définitions

- SheetMusicManagement : repo https://github.com/los-teoporos/sheet-music-management 
- Github Action : script éxécuté sur github, permettant d'automatiser certaines tâches, comme la génération des pdfs et des audios
- Site : le site web de Losteoporos, repo https://github.com/los-teoporos/website
- Google Drive : le google drive partagé contenant entre-autre les partitions, rangées par status
- Google Sheet : le google sheet permettant à la fanfare de s'organiser, qui contient aussi les notes des morceaux, les setlists, ...
- Musescores : fichiers contenant les partitions
- PDFs : partition des morceaux, et carnets pour chaque instrument
- Audios : audios MIDI généré à partir des Musescores
- Assets : PDFs + Audios

## Fonctionnement

### 1. Manuel : Google Drive

Aucun lien avec le status des morceaux dans le Google sheet, ni aucun lien avec le site affichant les notes, les PDFs, les Audios, les setlists.

Il faut avoir la configuration sur son ordinateur (styles, icones, etc) et les scripts pour générer les Assets avec musescore (accessible sur le repo SheetMusicManagement)

Ensuite, il faut uploader les fichiers modifiés sur le drive.

```mermaid
sequenceDiagram
    actor A as Alice
    participant GD@{ "type" : "database" } as Google Drive

    A->>A : Génère les Assets sur son ordi
    A->>GD : Upload les Assets
```

### 2. Semi-automatisé : Google Drive -> Github -> Site

Les Musescores et Assets sont sur le Google Drive. 
La Github Action du SheetMusicManagement génère les Assets, mais il est impossible de les upload automatiquement sur le Google Drive.

Ainsi, pour mettre à jour une partition sur le Site, il faut mettre à jour le Google Drive, attendre la génération sur SheetMusicManagement, puis upload manuellement les Assets sur le Google Drive.

```mermaid
sequenceDiagram
    actor A as Alice
    participant W as Website
    participant SMM as SheetMusicManager
    participant GD@{ "type" : "database" } as Google Drive

    A->>GD : Change un fichier, déplace un dossier, ...
    
    loop Chaque matin et chaque soir
        SMM->>GD : Télécharge les Musescores changés
        GD-->>SMM : Musescores
        SMM-->>SMM : Génère les Assets et les push sur le repo
        W->>SMM : Pull le dernier commit du repo
        SMM-->>W : Musescores+Assets
    end

    A->>GD : Upload manuel des Assets
```

### 3. Automatisé : Site <-> Github 

Tout se fait depuis le Site, les Musescores+Assets sont stockés sur le repo SheetMusicManagement qui permet:
- de conserver un historique des modifications
- de générer les Assets via la Github Action
- de garder une architecture de dossiers humainement lisible et utilisable sans le site

> [!NOTE]
> Modifier le status d'un morceau déplacera le dossier contentant Musescore+Assets dans le dossier correspondant au status et re-génèrera les carnets si besoin

```mermaid
sequenceDiagram
    actor A as Alice

    participant W as Website
    participant DB@{ "type" : "database" } as DB
    participant SMM as SheetMusicManager
    
    A->>W : Crée un morceau
    W-->>DB : Stocke le nom, le status, ...

    A->>+W : Upload le Musescore
    W-->>W : Stocke le Musescore
    Note right of W : Chemin:<br>/assets/{status}/{nom}/{nom}.mscz<br>Status : <br>- morceaux<br>- morceaux_morts<br>- morceaux_agonisants
    W->>+SMM : Push le Musescore
    
    SMM->>SMM : Génère les Assets et les push sur le repo

    SMM-->>-W : Averti que la génération est terminée via un WebHook
    
    W->>SMM : Pull le dernier commit du repo
    SMM-->>W : Musescores+Assets

    W-->>-A : Affiche les Assets à jour
```
