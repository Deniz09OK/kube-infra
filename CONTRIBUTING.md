# CONTRIBUTING.md — convention de commits

Ce repo fait partie du projet Epitech "KUBE — Sailing through the Clouds" (equipe SecurePods). Tous les commits doivent suivre ce format :

```
<type>(SEC-XX): <description courte>
```

- `<type>` : toujours en anglais (norme Conventional Commits), voir tableau ci-dessous
- `SEC-XX` : le numero du ticket Jira concerne (ex. `SEC-14`) — **obligatoire**, permet de tracer chaque commit jusqu'au ticket
- `<description courte>` : en francais, au present, sans majuscule en debut, sans point final

## Types autorises

| Type | Utilisation |
|---|---|
| `feat` | nouvelle fonctionnalite / nouveau manifest / nouvel outil deploye |
| `fix` | correction d'un bug ou d'une config cassee |
| `docs` | documentation uniquement (README, AGENT.md, Confluence lie, etc.) |
| `chore` | tache d'entretien, config, dependances |
| `refactor` | reorganisation sans changement de comportement |
| `test` | ajout ou correction de tests |
| `ci` | pipeline CI/CD |

## Exemples

```
feat(SEC-21): ajoute l'ingress controller traefik
fix(SEC-34): corrige le mapping groupe oidc vers rolebinding
docs(SEC-5): ajoute agent.md et claude.md
chore(SEC-12): met a jour la version du role ansible
feat(SEC-40): ajoute le cronjob de dump mysql
refactor(SEC-18): reorganise les overlays kustomize staging/prod
```

## Regle

- **Un commit = un ticket Jira.** Si un changement touche plusieurs tickets, faire plusieurs commits distincts plutot qu'un seul commit avec plusieurs scopes.
- Si un commit ne correspond a aucun ticket (ex. correction triviale, typo), utiliser `chore(divers): <description>` a defaut, mais rester exceptionnel — privilegier la creation d'un ticket si le travail est notable.

## Pas de co-authorship IA

Si un outil d'IA (Claude, Copilot, ChatGPT, etc.) a aide a ecrire du code ou de la doc, **ne pas** ajouter de ligne `Co-Authored-By: <IA>` ni de signature du type "Generated with <IA>" dans le message de commit. L'auteur du commit reste la personne qui a relu et valide le changement, quel que soit l'outil utilise pour le produire.
