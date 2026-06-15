# TrustGuard KYC

Microservice d'extraction et de validation automatique de documents d'identité (OCR + vérification métier) pour l'onboarding B2C des startups et scale-ups.

TrustGuard réduit le délai de validation de profil (actuellement 24–48 h) en automatisant le scoring de conformité utilisateur, tout en respectant les normes européennes.

## Fonctionnalité MVP (P0)

| Feature | Priorité | Description |
|---------|----------|-------------|
| **UserConformityScoring** | P0 | Scoring semi-automatisé de la conformité utilisateur à partir de documents d'identité |

## Stack

| Couche | Technologie |
|--------|-------------|
| API | Spring Boot |
| Frontend | Next.js |

## Documentation

Toute la documentation est versionnée dans ce dépôt ([Documentation as Code](docs/)).

| Ressource | Emplacement |
|-----------|-------------|
| Produit (PRD) | [docs/product/](docs/product/) |
| Architecture | [docs/architecture/](docs/architecture/) |
| Guides | [docs/guides/](docs/guides/) |
| Contrat API (OpenAPI) | [openapi.yml](openapi.yml) _(à venir)_ |

## Prérequis

- **Java 21+** (API Spring Boot)
- **Node.js 20+** et **npm** (frontend Next.js)
- **Docker** et **Docker Compose** (environnement local recommandé)
- **Git**

## Installation

> Le boilerplate Spring Boot et le frontend Next.js seront ajoutés dans les prochaines itérations du MVP.

```bash
git clone https://github.com/steevnPrm/TrustGuard-KYC.git
cd TrustGuard-KYC
```

## Lancement (local)

_Instructions détaillées à compléter une fois les modules `api/` et `web/` en place._

```bash
# API (Spring Boot) — à venir
# cd api && ./mvnw spring-boot:run

# Frontend (Next.js) — à venir
# cd web && npm install && npm run dev
```

## Structure du dépôt

```
TrustGuard-KYC/
├── README.md          # Ce fichier
├── openapi.yml        # Contrat API OpenAPI (à venir)
├── api/               # Microservice Spring Boot (à venir)
├── web/               # Application Next.js (à venir)
└── docs/              # Documentation as Code
    ├── product/       # PRD, périmètre MVP
    ├── architecture/  # System design, domain design, ADR
    └── guides/        # Installation, dev local, contrat API
```

## Licence

Propriétaire — licence fermée, tous droits réservés.
