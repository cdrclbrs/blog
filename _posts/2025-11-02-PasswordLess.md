---
layout: post
title: "Identités fédérées et passwordless : la nouvelle ère de l’authentification"
excerpt: "De la fédération d’identité au zéro mot de passe, une approche standardisée et résistante au phishing."
excerpt_separator: "<!--more-->"
categories:
  - Security
tags:
  - passwordless
  - mfa
last_modified_at: 2025-11-02T02:29:12-11:00
---

# Bien comprendre la mécanique d’une authentification par mot de passe

Quand vous vous connectez, vous fournissez un identifiant et un mot de passe. Rien de neuf , tout le monde connait deja le célèbre « admin/1234 »… 
Le serveur, lui, ne garde pas votre secret en clair. Il stocke un hash du mot de passe, calculé via une fonction de dérivation type bcrypt, scrypt etc, et surtout avec un sel (salt)
À la connexion, il refait le calcul et compare les hash: Si ça matche, la connexion est authorisée.
Les bons systèmes ajoutent du rate limiting, du captcha et des verrous progressifs pour freiner les robots. 
Il est aussi possible de protèger les pages d'authen avec des WAP comme Cloudflare.

<p align="center">
  <img src="https://blog.lbrs.io/images/weak.jpg" alt="weak" style="width:30%;">
</p>

Le problème, c’est que tout repose sur un secret mémorisé.
Donc c’est sensible au phishing (faux site), au bruteforce et au credential stuffing.
Et quand une base de passwords fuite, (c'est a dire toutes les semaines🤡), des attaquants peuvent tenter des attaques hors-ligne.
Les bonnes pratiques aident (mots de passe longs, dérivations lentes, sel systématique), mais ça ne supprime pas les faiblesses structurelles. D’où la généralisation du fameux MFA.

<p align="center">
  <img src="https://blog.lbrs.io/images/MFAwouldbegreat.jpg" alt="mfa would be great" style="width:60%;">
</p>

# Ceinture et bretelles : le MFA

Le Multi-Factor Authentication consiste à prouver son identité avec au moins deux catégories différentes parmis 3 groupes : ce que l’on **sait** (mot de passe, PIN), ce que l’on **a** (téléphone, clé physique), ce que l’on **est** (biométrie). L’idée est simple : si un facteur tombe, l’attaquant n’a pas gagné pour autant. Le principe de multi - couches.

En pratique : mot de passe + code OTP, ou mot de passe + notification “push”, ou encore biométrie + clé matérielle.
comment ca fonctionne : On entre le mot de passe, puis défi MFA, puis validation, puis émission d’une session

<p align="center">
  <img src="https://blog.lbrs.io/images/leMFA.png" alt="mfa would be great" style="width:60%;">
</p>


On dort un peu mieux, mais pas totalement : 
un OTP peut être volé en temps réel via un proxy de phishing type **EvilGinx**, un SMS peut être détourné (SIM-swap), et les « push » peuvent être approuvés par fatigue. 

Moralité : mieux que rien, souvent indispensable, mais pas à l’épreuve de tout. hey oui, et beaucoup trop de gens pensent le contraire....et se pensent souvent hors d'atteinte avec le MFA: mauvaise idée. Il existe des tas de manières de contourner cela.

<p align="center">
  <img src="https://blog.lbrs.io/images/mfafails.jpg" alt="mfa failst" style="width:60%;">
</p>


## Ce qui se passe après le login : sessions et jetons

Une fois authentifié, le site ne vous redemande pas votre mot de passe à chaque clic.
Il vous donne un cookie de session que vous ne voyez jamais (signé/chiffré, flags HttpOnly, Secure, SameSite), qui prouve au serveur que « c’est toujours vous »

Dans des architectures plus évoluées, surtout avec un SSO, on parle de jetons :C'est un access token (pour accéder aux API) et ID token (pour transporter l’assertion d’identité).
Ces jetons ne sont pas des facteurs MFA. 
Ils matérialisent le résultat d’une authentification réussie et doivent être protégés comme votre wallet BTC :D! On ne laisse pas traîner ses clés de voiture sur le capot.

## Point sur les identités fédérées et SSO

Quand plusieurs applications doivent reconnaître le même user, on veut du SSO. La federation d'identité ca a plus de 20 ans c'est pas nouveau, mais c'est important de bien comprendre le fonctionnement car ce n'est pas toujours limpide pour pas mal de monde même dans le domaine IT.
C'est donc un service central d'Identity provider (IdP) qui authentifie l’utilisateur et émet une assertion ou un jeton que les applications (services providers) acceptent.
Le standard historique, c’est le fameux SAML, surtout côté entreprises (SSO interne, partenaires). Les environnements modernes s’appuient plutôt sur OAuth 2.0 et OpenID Connect, plus adaptés au web et aux API.

En résumé, chaque standard a son rôle :

SAML est danns toutes les entreprises : basé sur XML et pensé pour le web des années 2000, il reste très utilisé pour le SSO interne et les portails partenaires. Solide, mais un peu lourd ( vous avez déja du gérer une archi redondante de federation ADFS avant Entra ID? :)

OAuth 2.0 a changé la donne en permettant à une application d’accéder à vos données sans jamais voir votre mot de passe. C’est la fameuse option « Se connecter avec Google » ou « avec Facebook ». Par contre, OAuth ne gère pas directement l’identité, seulement l’autorisation d’accès.

OpenID Connect (OIDC) vient compléter OAuth : il ajoute la couche “qui suis-je ?” via un ID Token, une preuve standardisée de l’identité de l’utilisateur.

Ce modèle repose sur la confiance entre services : un site délègue l’authentification à un fournisseur d’identité, sans manipuler de mot de passe. C’est le même principe que les passkeys, mais à l’échelle du web et des organisations : l’identité est validée de façon forte, sans secret partagé ni risque de phishing.

## Petit refresh sur les OTP, TOTP et HOTP 

OTP signifie « One-Time Password ». Le serveur et vous partagez un secret. 
Vous ne l’envoyez jamais : vous envoyez un code dérivé une seule fois.
De son côté, le serveur recalcule le même code et compare.
C’est de la crypto pratique qui évite d’exposer le secret lui-même. 

### HOTP 

Avec HOTP, le code dépend d’un compteur: À chaque génération, le compteur s’incrémente.
Le serveur garde une fenêtre de tolérance pour se resynchroniser si vous avez généré des codes non utilisés.
Avantage : pas besoin d’horloge.
Inconvénient : la synchro peut dérailler, et il existe toujours un secret partagé des deux côtés.

### TOTP 

TOTP remplace le compteur par le temps (tranches de 30 s, typiquement)
Le code expire vite, ce qui est intuitif pour l’utilisateur. Il suffit d’une horloge à peu près correcte. Mais on dépend toujours d’un secret partagé et le phishing en temps réel reste possible : tapez le code au mauvais endroit come sur le site de l'attaquant, et quelqu’un peut le relayer instantanément.

## Les vecteurs d'attaque


Phishing par proxy : un attaquant peut intercepter et rejouer le code en direct, (CC evilGinx)
SMS fragile : le télécom et le SIM-swap ne sont pas vos amis
le Secret est côté serveur : risques de fuite.


##  Parlons standard FIDO2/WebAuthn : la crypto asymétrique passe à l’échelle du Web

Au lieu d’un secret réutilisable côté serveur, on utilise une paire de clés : une privée qui reste dans l’authentificateur (appareil, Secure Enclave/TPM, ou clé USB/NFC type Yubico), et une publique stockée par le service.
À l’enrôlement, l’authentificateur génère ces clés pour un site donné. 
À la connexion, il signe un challenge et inclut l’origin. Le serveur vérifie la signature avec la clé publique.
Pas de mot de passe, pas d’OTP, et surtout **une preuve liée au domaine**.
Un site de phishing ne peut pas obtenir une signature valable pour le vrai site. Simple, efficace.

### FIDO 2 
Fido2 est un ensemble de standarts de l'aliance FIDO et de W3C, il regourpe 2 composants principaux:  WetAuth, l’API côté navigateur pour créer et utiliser les identifiants publics et le protocole entre le navigateur/OS et l’authentificateur (clé matérielle ou authentificateur intégré) qui est le CTAP (Client To Authenticator Protocol).

FIDO2 = WebAuthn + CTAP.

<p align="center">
  <img src="https://blog.lbrs.io/images/fido2.jpg" alt="fido2" style="width:40%;">
</p>


Lors de la connection, le serveur envoie un challenge que l’appareil signe avec la clé privée. Vous validez localement (biométrie ou PIN). 
Puis, l’authentificateur signe le challenge, lié à l’origin. Le serveur vérifie et c’est terminé. Pas de mot de passe, pas d’OTP, pas de « code reçu par SMS ». C’est tres rapide. 

### Pourquoi c’est vraiment résistant au phishing ?

La signature ne vaut que pour le domaine exact.
Le faux site « pawn.me » n’obtiendra jamais une preuve valable pour « secure.com ». Et comme il n’y a pas de secret serveur à voler, les fuites massives de passwords ne sont plus un problème. WIN-WIN


# Passkeys et clés matérielles 

Une passkey est une version simplifiée et synchronisable de l’authentification FIDO2.
Votre appareil (OS ou clé) génère et stocke une paire de clés cryptographiques pour chaque site : la clé privée reste sur l’appareil, la clé publique va au serveur.
Quand vous vous connectez, l’appareil signe la demande après vérification locale (empreinte, visage, PIN).
Les passkeys peuvent être synchronisées via votre compte Apple, Google ou Microsoft, ce qui permet de vous reconnecter facilement sur tous vos appareils sans jamais manipuler de mot de passe

### YubiKey et autres clés « roaming »

<p align="center">
  <img src="https://blog.lbrs.io/images/yubico.png" alt="Yubi" style="width:60%;">
</p>


Les clés matérielles USB/NFC gardent la clé privée hors cloud, donc sur votre clef physique
Idéales pour les comptes sensibles et les administrateurs.
L’usage est simple (on touche la clé, on entre un PIN local si requis), mais il faut gérer l’inventaire, prévoir les pertes et fournir deux clés par personne critique. Ce n’est pas glamour, mais ça sécurise proprement. Le fait de toucher physiquement la clef garanti qu'une automatisation ne pourra pas "utiliser"la clef privée de votre clef physique.

## Le passwordless

**Passwordless** signifie : authentifier un utilisateur sans secret mémorisé côté humain.
La référence robuste, c’est FIDO2/WebAuthn : preuve cryptographique, liée au domaine, avec vérification locale.
lorsque vous cliquez « Se connecter avec une passkey », vous validez avec votre empreinte ou votre visage, votre appareil signe le challenge (du bon domaine), et c’est réglé.

### Concrêtement, cela apporte quoi par rapport au simple password+MFA ?

Résistance au phishing par conception.
Plus de base de mots de passe à protéger.
Moins de friction : pas de code à recopier, moins de resets.
Standard ouvert bien supporté, pilotable côté IAM (politiques , audit).

### Les points d’attention 

Les passkeys synchronisées demandent des politiques solides de récupération de compte et de sécurité des appareils.
Les clés matérielles impliquent de la logistique et une gouvernance claire.
Côté produit, il faut activer WebAuthn, travailler sur l’enrôlement et prévoir des chemins de secours. 
<p align="center">
  <img src="https://blog.lbrs.io/images/fpasskey.jpg" alt="PassKeyArg" style="width:40%;">
</p>


# En synthèse

Le passwordless n’est pas un gadget UX. 
C’est une réduction massive du risque : plus de mot de passe à voler, des preuves non rejouables, liées au domaine. 
Et le bonus non négligeable : moins de tickets Jira « Maman, j’ai perdu mon mot de passe »