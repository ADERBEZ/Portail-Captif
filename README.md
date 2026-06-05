[Uploading portailv1.01.html…]()
# Portail-Captif

*Le principe du portail captif est de permettre à une personne de se connecter à réseau Wi-Fi, sous conditions de fournir des identifiants (e-mail, nom, prénom…).*

> Ce GitHub ne représente uniquement la partie Web (visuel) du Portail Actif.

## Modification v1 -> v1.01
Le `<div id="login-form">` est devenu `<form method="post" action="$PORTAL_ACTION$">` qui englobe maintenant toute la zone (identifiant, mot de passe, charte, case à cocher et bouton). 
J'ai ajouté les champs cachés `redirurl` et `zone`, renommé les champs en `auth_user` / `auth_pass`, transformé le bouton en `type="submit" name="accept" value="Continue"`, et supprimé la fausse simulation (`setTimeout`) qui affichait toujours une erreur. 

Le JavaScript ne fait plus que la vérification des champs vides + charte cochée : si tout est bon, il laisse pfSense traiter réellement l'authentification.


Voici le [Code PC v1.01](https://drive.google.com/file/d/10qwJW5rhb8Wuup7cCVtwODqpnuOhXBMn/view?usp=sharing).
