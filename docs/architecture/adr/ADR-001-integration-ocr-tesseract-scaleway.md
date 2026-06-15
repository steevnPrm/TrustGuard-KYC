# ADR-001 — Intégration OCR Tesseract dockerisé sur Scaleway

**Date :** 2026-06-15  
**Statut :** Accepté  
**Contexte implémentation :** Étape `UserVerification` — remplacement de Mindee  
**Remplace :** [ADR-013 — Intégration OCR Mindee](ADR-013-integration-ocr-mindee.md) (vision dépôt historique)

## Contexte

L'[ADR-013](ADR-013-integration-ocr-mindee.md) (dépôt historique) retenait **Mindee** comme provider OCR pour l'extraction structurée des pièces d'identité (CNI, passeport) dans l'étape `UserVerification`.

Cette option présente des contraintes incompatibles avec le périmètre **MVP / démo / recherche de stage** :

| Contrainte Mindee | Impact |
|-------------------|--------|
| Coût récurrent (~44 €/mois et au-delà) | Bloquant pour un projet portfolio sans budget |
| Sous-traitant RGPD externe | DPA obligatoire (TRU-13), complexité juridique disproportionnée en phase démo |
| Dépendance SaaS | Vendor lock-in sur un composant remplaçable |

TrustGuard cible des **solutions hébergées en Union européenne** avec un positionnement souverain ([ADR-004](ADR-004-hebergement-cloud-scaleway.md)). L'OCR doit donc s'inscrire dans le même périmètre d'hébergement que l'API et la base de données, **sans transfert vers un SaaS tiers**.

**Tesseract** (moteur OCR open source, Apache 2.0) déployé en **conteneur Docker sur Scaleway** permet un traitement auto-hébergé, sans coût de licence, aligné avec l'infrastructure retenue en Cycle 1.

## Décision

Remplacer **Mindee** par **Tesseract 5.x** déployé comme **microservice conteneurisé** sur **Scaleway Serverless Containers** (ou **Scaleway Instances** en alternative), région **`fr-par` (Paris)**.

### Architecture retenue

```
[API Spring Boot — Scaleway fr-par]
       │  PhoneCodeAccepted → événement UserVerification
       ▼
[Worker EDA — Scaleway fr-par]
       │  HTTP interne (réseau privé / VPC)
       ▼
[Service OCR Tesseract — Scaleway fr-par]
       ├─ Image Docker (Tesseract + wrapper HTTP minimal)
       ├─ Pas d'exposition publique
       └─ Pas de sortie réseau vers SaaS tiers
       ▼
[UserVerified] → UserAnalyze
```

### Contrat technique

| Élément | Choix |
|---------|-------|
| Moteur OCR | Tesseract 5.x (`fra` + `eng` language packs) |
| Packaging | Image Docker custom (Tesseract + wrapper HTTP minimal, ex. FastAPI ou service Java léger) |
| Orchestration | Scaleway Serverless Containers (région `fr-par`) |
| Registry | Scaleway Container Registry (`fr-par`) |
| Réseau | Communication interne depuis le worker Spring ; **endpoint non exposé sur Internet** |
| Pattern applicatif | Port hexagonal : interface `OcrProvider` |
| Hébergement | Aligné [ADR-004](ADR-004-hebergement-cloud-scaleway.md) — même cloud, même région |

### Implémentations `OcrProvider`

| Environnement | Implémentation | Rôle |
|---------------|----------------|------|
| `local` / `test` | `MockOcrProvider` | Développement sans infra Scaleway |
| `local` (optionnel) | `TesseractDockerProvider` | `docker-compose` avec la même image OCR |
| `staging` / `prod` | `TesseractScalewayProvider` | Appel au service conteneurisé via endpoint privé |

### Flux `UserVerification` (inchangé fonctionnellement)

| Étape | Action |
|-------|--------|
| 1 | `PhoneCodeAccepted` → déclenchement `UserVerification` (EDA) |
| 2 | Worker récupère `UserDocument` depuis le stockage interne ([ADR-010](ADR-010-stockage-documents-mvp-en-base.md)) |
| 3 | Appel HTTP au service Tesseract → extraction texte brut + parsing métier côté worker |
| 4 | Publication `UserVerified` → enchaînement `UserAnalyze` |
| 5 | **Aucun sous-traitant OCR externe** — pas de DPA Mindee requis pour cette brique |

### Parsing post-OCR

Tesseract retourne du **texte brut**. Le mapping vers un objet domaine `ExtractedIdentity` (nom, prénom, date de naissance, n° document) est réalisé par un **`IdentityDocumentParser`** côté worker Spring (regex / heuristiques sur CNI et passeport français).

> **Compromis assumé :** précision inférieure à Mindee sur les pièces d'identité. Acceptable pour le MVP ; une intégration SaaS spécialisée reste envisageable en phase ultérieure via le même port `OcrProvider`.

## Conséquences

### Positives

- **Coût OCR : 0 €** de licence (hors compute Scaleway, marginal en démo)
- **Pas de transfert vers un SaaS OCR tiers** — documents traités sur infrastructure Scaleway Paris
- **Cohérence infra** — API, base PostgreSQL ([ADR-005](ADR-005-persistance-postgresql-scaleway.md)) et OCR sur le même fournisseur UE
- **Pas de DPA** avec un éditeur OCR externe (Mindee)
- **Remplaçabilité** : `OcrProvider` permet de basculer vers Mindee ou un autre moteur sans refonte du domaine
- **Aligné EDA** : latence OCR variable (≥ 2 s) reste asynchrone ([ADR-006](ADR-006-router-communication-soa-eda.md))
- **Reproductibilité** : même image Docker en local (`docker-compose`) et en prod (Scaleway)

### Négatives / compromis

| Compromis | Détail |
|-----------|--------|
| Précision OCR | Tesseract générique < Mindee spécialisé ID ; taux d'erreur plus élevé sur CNI/passeport |
| Parsing maison | `IdentityDocumentParser` à maintenir (fragile selon qualité photo/scan) |
| Charge ops | Gestion image Docker, déploiement conteneur, monitoring — plus lourd qu'un appel API Mindee |
| Pas « solution française » | Tesseract est un OSS international, pas un éditeur FR (argument commercial différent de Mindee) |
| Pas de score de confiance natif | Mindee fournit des scores par champ ; Tesseract nécessite un scoring maison ou heuristique |

### Mitigations

- `MockOcrProvider` pour les tests et la démo sans Scaleway
- Seuils de confiance basés sur la complétude des champs extraits (scoring métier TrustGuard)
- Dossiers en statut `PENDING_REVIEW` si parsing incomplet (filet de sécurité)
- Réintroduction ultérieure d'un provider SaaS spécialisé derrière `OcrProvider` si la précision devient bloquante en prod

## Hors scope

- Matching document ↔ déclaration utilisateur
- Décision automatique KYC
- Changement de cloud provider (Scaleway reste le choix retenu — [ADR-004](ADR-004-hebergement-cloud-scaleway.md))

## Références

- [ADR-004 — Hébergement cloud Scaleway (UE)](ADR-004-hebergement-cloud-scaleway.md)
- [ADR-005 — Persistance PostgreSQL sur Scaleway](ADR-005-persistance-postgresql-scaleway.md)
- [ADR-006 — Router communication SOA / EDA](ADR-006-router-communication-soa-eda.md)
- [ADR-010 — Stockage documents MVP en base](ADR-010-stockage-documents-mvp-en-base.md)
- [ADR-013 — Intégration OCR Mindee](ADR-013-integration-ocr-mindee.md) *(remplacé)*
- [ADR-020 — Machine à états inscription](ADR-020-machine-etats-inscription.md)
- [ADR-022 — Séquence SMS puis parallèle email et OCR](ADR-022-sequence-sms-puis-email-ocr.md)
- [Tesseract OCR](https://github.com/tesseract-ocr/tesseract)
- [Scaleway Serverless Containers](https://www.scaleway.com/en/serverless-containers/)
