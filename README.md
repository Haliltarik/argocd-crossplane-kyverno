
# Démo : Garantir la Conformité des Buckets S3 avec Kyverno

---

## 🎯 SCHÉMA SIMPLE 

```
┌─────────────────────────────────────────────────────────────────────────────┐
│         GARANTIR LA CONFORMITÉ DES BUCKETS S3 AVEC KYVERNO                  │
└─────────────────────────────────────────────────────────────────────────────┘


    👨‍💻 DÉVELOPPEUR                           👨‍💻 DÉVELOPPEUR
       Crée BucketClaim                        Crée BucketClaim
            │                                       │
            │                                       │
   ┌────────▼──────────┐                   ┌────────▼──────────┐
   │ requireSSL: true  │                   │ requireSSL: false │
   │ encryption: true  │                   │ encryption: false │
   └────────┬──────────┘                   └────────┬──────────┘
            │                                       │
            │                                       │
            ▼                                       ▼
   ┌─────────────────────┐             ┌─────────────────────┐
   │      KYVERNO        │             │      KYVERNO        │
   │   Policy Engine     │             │   Policy Engine     │
   │                     │             │                     │
   │   ✅ Validation OK  │             │  ❌ BLOQUÉ !       │
   └────────┬────────────┘             └──────────────────────┘
            │                                      │
            │                                      │
            ▼                                      ▼
   ┌─────────────────────┐             ┌──────────────────────┐
   │    CROSSPLANE       │             │   Admission Denied   │
   │  Injecte Sécurité   │             │                      │
   │  • TLS 1.2 min      │             │  Message clair :     │
   │  • HTTPS only       │             │  "requireSSL et      │
   │  • Encryption       │             │   encryption requis" │
   └────────┬────────────┘             └──────────────────────┘
            │
            │
            ▼
   ┌─────────────────────┐
   │      AWS S3         │
   │  ✅ Bucket Sécurisé │
   │  • 4 Policy Stmts   │
   │  • Chiffré          │
   │  • Accès bloqué     │
   └─────────────────────┘


   ✅ CONFORMITÉ GARANTIE              ❌ NON-CONFORMITÉ IMPOSSIBLE
   ✅ Self-service sécurisé            ❌ Blocage avant création
   ✅ 100% auditable                   ❌ Feedback immédiat
```

---

## Version Alternative (Plus Visuelle)

```
┌──────────────────────────────────────────────────────────────────────┐
│    SÉCURISATION S3 : VALIDATION AVANT CRÉATION (SHIFT-LEFT)          │
└──────────────────────────────────────────────────────────────────────┘


                    👨‍💻 DÉVELOPPEUR
                         │
                         │ Commit BucketClaim
                         │ dans Git
                         ▼
                    ┌─────────┐
                    │   GIT   │
                    │ + ArgoCD│
                    └────┬────┘
                         │
            ┌────────────┴────────────┐
            │                         │
            ▼                         ▼
    ┌──────────────┐          ┌──────────────┐
    │ requireSSL:  │          │ requireSSL:  │
    │   true ✅    │          │  false ❌   │
    └──────┬───────┘          └──────┬───────┘
           │                         │
           ▼                         ▼
    ┌─────────────────────────────────────────┐
    │         🛡️  KYVERNO                     │
    │      (Policy Enforcement)               │
    └─────────────────────────────────────────┘
           │                         │
           ▼                         ▼
    ✅ VALIDÉ                 ❌ BLOQUÉ
           │                         │
           ▼                         ▼
    ┌──────────────┐          ┌──────────────┐
    │  CROSSPLANE  │          │   FEEDBACK   │
    │  Provisionne │          │  "Corrigez   │
    │              │          │  le claim"   │
    └──────┬───────┘          └──────────────┘
           │
           ▼
    ┌──────────────┐
    │   AWS S3     │
    │ Bucket avec: │
    │ • TLS 1.2+   │
    │ • HTTPS      │
    │ • Chiffré    │
    └──────────────┘


    💡 RÉSULTAT : 100% conformité • 0 incident • Self-service sécurisé
```

---

## Version Minimaliste (Encore plus simple)

```
┌────────────────────────────────────────────────────────────┐
│     KYVERNO : VALIDATION AVANT CRÉATION                    │
└────────────────────────────────────────────────────────────┘


        ✅ CONFORME                  ❌ NON CONFORME
        ───────────                  ───────────────

    requireSSL: true           requireSSL: false
    encryption: true           encryption: false
           │                           │
           ▼                           ▼
    ┌─────────────┐            ┌─────────────┐
    │   KYVERNO   │            │   KYVERNO   │
    │     ✅      │            │     ❌     │
    └──────┬──────┘            └──────┬──────┘
           │                           │
           ▼                           ▼
    CROSSPLANE                   BLOQUÉ
    provisionne                  
           │                           
           ▼                           
    Bucket S3                          
    sécurisé                           


    🎯 MESSAGE CLÉ : Impossible de créer des buckets non conformes
```

---


**Utilisez le 2ème schéma (Version Alternative)** car il :
- ✅ Tient sur 1 slide
- ✅ Montre clairement les 2 scénarios (conforme/non-conforme)
- ✅ Est visuel et facile à comprendre
- ✅ Convient aux managers ET aux DevOps
- ✅ Montre le flux complet (Git → Kyverno → Crossplane → AWS)

**Couleurs suggérées pour PowerPoint :**
- 🟢 Vert : Flux conforme (✅)
- 🔴 Rouge : Flux bloqué (❌)
- 🔵 Bleu : Kyverno (brique centrale)
- ⚫ Gris : Git, ArgoCD, Crossplane, AWS


🧪 Tests de Validation
Test 1 : Création directe de Bucket (Doit être bloqué)
bash
cat <<EOF | kubectl apply -f -
apiVersion: s3.aws.crossplane.io/v1beta1
kind: Bucket
metadata:
  name: test-insecure-bucket
spec:
  forProvider:
    acl: public-read
    locationConstraint: eu-west-1
  providerConfigRef:
    name: default
EOF
Résultat attendu : ❌ Bloqué par block-direct-bucket-creation

Test 2 : Création de Composition custom (Doit être bloqué)
bash
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: my-insecure-composition
spec:
  compositeTypeRef:
    apiVersion: s3.aws.engie.org/v1alpha1
    kind: XBucket
  mode: Pipeline
  pipeline:
    - step: bucket
      functionRef:
        name: crossplane-contrib-function-patch-and-transform
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1
        kind: Resources
        resources:
          - name: bucket
            base:
              apiVersion: s3.aws.crossplane.io/v1beta1
              kind: Bucket
              spec:
                forProvider:
                  acl: public-read
                  locationConstraint: eu-west-1
                providerConfigRef:
                  name: default
EOF
Résultat attendu : ❌ Bloqué par block-unauthorized-compositions

Test 3 : BucketClaim avec composition non approuvée (Doit être bloqué)
bash
cat <<EOF | kubectl apply -f -
apiVersion: s3.aws.engie.org/v1alpha1
kind: BucketClaim
metadata:
  name: test-evil-bucket
  namespace: crossplane-system
spec:
  bucketName: test-evil-bucket
  location: eu-west-1
  requireSSL: true
  encryption: true
  compositionRef:
    name: fake-composition  # ❌ Pas dans la whitelist
  tags:
    - key: Environment
      value: test
EOF
Résultat attendu : ❌ Bloqué par enforce-approved-compositions-only

Test 4 : BucketClaim valide (Doit fonctionner)
bash
cat <<EOF | kubectl apply -f -
apiVersion: s3.aws.engie.org/v1alpha1
kind: BucketClaim
metadata:
  name: test-valid-bucket
  namespace: crossplane-system
spec:
  bucketName: test-valid-bucket-$(date +%s)
  location: eu-west-1
  requireSSL: true
  encryption: true
  compositionRef:
    name: s3bucket-secure  # ✅ Dans la whitelist
  tags:
    - key: Environment
      value: test
    - key: Owner
      value: platform-team
EOF
Résultat attendu : ✅ Accepté par Kyverno → Crossplane crée le bucket sécurisé


**Titre du slide :**
"Kyverno : Garantir la Conformité AVANT la Création"

**Sous-titre :**
"Validation automatique • Blocage des non-conformités • Self-service sécurisé"
