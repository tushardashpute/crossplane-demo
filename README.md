# Complete Crossplane Example: Database Provisioning

This example demonstrates all Crossplane components working together to provision a database.

## Architecture Overview

```
User creates CR (Claim) → XRD defines API → Composition renders resources → Function processes → MRs created
```

---

## 1. XRD (Composite Resource Definition)

**Purpose**: Defines the custom API schema for your infrastructure

**File**: `definition.yaml`

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.example.com
spec:
  group: example.com
  names:
    kind: XDatabase
    plural: xdatabases
  # Cluster-scoped composite resource
  claimNames:
    kind: Database
    plural: databases
  # Namespace-scoped claim
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                parameters:
                  type: object
                  properties:
                    size:
                      type: string
                      enum: [small, medium, large]
                      description: "Database size"
                    storageGB:
                      type: integer
                      minimum: 20
                      maximum: 1000
                    version:
                      type: string
                      default: "14"
                    environment:
                      type: string
                      enum: [dev, staging, prod]
                  required:
                    - size
                    - environment
              required:
                - parameters
            status:
              type: object
              properties:
                endpoint:
                  type: string
                  description: "Database connection endpoint"
                ready:
                  type: boolean
                username:
                  type: string
```

**Explanation**:
- Defines `XDatabase` (cluster-scoped) and `Database` (namespace-scoped claim)
- Specifies input parameters users can provide
- Defines status fields that will be populated after provisioning

---

## 2. Composition

**Purpose**: Defines HOW to create infrastructure resources

**File**: `composition.yaml`

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: database-aws
  labels:
    provider: aws
spec:
  compositeTypeRef:
    apiVersion: example.com/v1alpha1
    kind: XDatabase
  
  mode: Pipeline
  
  pipeline:
    # Step 1: Process inputs and prepare context
    - step: prepare-context
      functionRef:
        name: function-go-templating
      input:
        apiVersion: gotemplating.fn.crossplane.io/v1beta1
        kind: GoTemplate
        source: Inline
        inline:
          template: |
            {{ $xr := .observed.composite.resource }}
            {{ $size := $xr.spec.parameters.size }}
            {{ $env := $xr.spec.parameters.environment }}
            
            # Determine instance size based on input
            {{- $instanceSize := "" }}
            {{- if eq $size "small" }}
              {{- $instanceSize = "db.t3.micro" }}
            {{- else if eq $size "medium" }}
              {{- $instanceSize = "db.t3.medium" }}
            {{- else }}
              {{- $instanceSize = "db.m5.large" }}
            {{- end }}
            
            # Generate unique name
            {{ $dbName := printf "db-%s-%s" $env $xr.metadata.name }}
            
            ---
            apiVersion: meta.gotemplating.fn.crossplane.io/v1alpha1
            kind: Context
            data:
              instanceSize: {{ $instanceSize }}
              dbName: {{ $dbName }}
              storageGB: {{ $xr.spec.parameters.storageGB | default 100 }}
              version: {{ $xr.spec.parameters.version }}
    
    # Step 2: Render AWS resources
    - step: render-resources
      functionRef:
        name: function-go-templating
      input:
        apiVersion: gotemplating.fn.crossplane.io/v1beta1
        kind: GoTemplate
        source: Inline
        inline:
          template: |
            {{ $xr := .observed.composite.resource }}
            
            ---
            # Managed Resource 1: RDS Instance
            apiVersion: rds.aws.upbound.io/v1beta1
            kind: Instance
            metadata:
              name: {{ .context.dbName }}
              annotations:
                {{ setResourceNameAnnotation "rds-instance" }}
            spec:
              forProvider:
                region: us-east-1
                instanceClass: {{ .context.instanceSize }}
                engine: postgres
                engineVersion: {{ .context.version | quote }}
                allocatedStorage: {{ .context.storageGB }}
                dbName: {{ .context.dbName }}
                username: admin
                passwordSecretRef:
                  name: {{ printf "%s-password" .context.dbName }}
                  namespace: crossplane-system
                  key: password
                skipFinalSnapshot: true
                publiclyAccessible: false
              providerConfigRef:
                name: default
            
            ---
            # Managed Resource 2: Security Group
            apiVersion: ec2.aws.upbound.io/v1beta1
            kind: SecurityGroup
            metadata:
              name: {{ printf "%s-sg" .context.dbName }}
              annotations:
                {{ setResourceNameAnnotation "security-group" }}
            spec:
              forProvider:
                region: us-east-1
                description: Security group for database
                vpcId: vpc-12345678
                ingress:
                  - fromPort: 5432
                    toPort: 5432
                    protocol: tcp
                    cidrBlocks:
                      - 10.0.0.0/16
              providerConfigRef:
                name: default
            
            ---
            # Managed Resource 3: Kubernetes Secret with credentials
            apiVersion: kubernetes.crossplane.io/v1alpha2
            kind: Object
            metadata:
              name: {{ printf "%s-secret" .context.dbName }}
              annotations:
                {{ setResourceNameAnnotation "k8s-secret" }}
            spec:
              forProvider:
                manifest:
                  apiVersion: v1
                  kind: Secret
                  metadata:
                    name: {{ printf "%s-connection" .context.dbName }}
                    namespace: {{ $xr.spec.claimRef.namespace | default "default" }}
                  type: Opaque
                  stringData:
                    endpoint: {{ printf "%s.rds.amazonaws.com:5432" .context.dbName }}
                    username: admin
                    database: {{ .context.dbName }}
              providerConfigRef:
                name: default
            
            ---
            # Update XR status with connection info
            apiVersion: example.com/v1alpha1
            kind: XDatabase
            status:
              endpoint: {{ printf "%s.rds.amazonaws.com:5432" .context.dbName }}
              ready: {{ eq (index .observed.resources "rds-instance" "resource" "status" "conditions" 0 "status") "True" }}
              username: "admin"
    
    # Step 3: Mark resources as ready automatically
    - step: auto-ready
      functionRef:
        name: function-auto-ready
```

**Explanation**:
- Pipeline mode with 3 steps
- Step 1: Processes user input, calculates values
- Step 2: Renders actual cloud resources (RDS, SecurityGroup, Secret)
- Step 3: Automatically marks resources ready when conditions met
- Uses templating to dynamically generate resources based on input

---

## 3. Functions

**Purpose**: Process and transform resources in the pipeline

### Function 1: function-go-templating
**What it does**: Templating engine using Go templates

### Function 2: function-auto-ready
**What it does**: Automatically marks composed resources as ready

**Installation** (if needed):
```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-go-templating
spec:
  package: xpkg.upbound.io/crossplane-contrib/function-go-templating:v0.4.0
```

---

## 4. CR (Composite Resource / Claim)

**Purpose**: User's request to create infrastructure

### Option A: Namespace-scoped Claim (typical for developers)

**File**: `database-claim.yaml`

```yaml
apiVersion: example.com/v1alpha1
kind: Database
metadata:
  name: my-app-db
  namespace: production
spec:
  parameters:
    size: medium
    storageGB: 100
    version: "14"
    environment: prod
  # Optional: Select specific composition
  compositionSelector:
    matchLabels:
      provider: aws
  # How to handle deletion
  compositionUpdatePolicy: Automatic
  # Where to write connection details
  writeConnectionSecretToRef:
    name: my-app-db-connection
```

### Option B: Cluster-scoped Composite Resource

**File**: `xdatabase.yaml`

```yaml
apiVersion: example.com/v1alpha1
kind: XDatabase
metadata:
  name: my-app-db-xyz123
spec:
  parameters:
    size: large
    storageGB: 500
    version: "15"
    environment: prod
  compositionRef:
    name: database-aws
```

**Explanation**:
- Claims are namespace-scoped (used by app teams)
- Composite Resources are cluster-scoped (used by platform teams)
- Crossplane creates XR automatically when you create a Claim

---

## 5. MR (Managed Resources)

**Purpose**: Actual cloud provider resources created by Crossplane

These are automatically created by the Composition. Examples:

### MR 1: RDS Instance
```yaml
apiVersion: rds.aws.upbound.io/v1beta1
kind: Instance
metadata:
  name: db-prod-my-app-db
  ownerReferences:
    - apiVersion: example.com/v1alpha1
      kind: XDatabase
      name: my-app-db-xyz123
      uid: ...
spec:
  forProvider:
    region: us-east-1
    instanceClass: db.m5.large
    engine: postgres
    # ... configuration from composition
status:
  conditions:
    - type: Ready
      status: "True"
  atProvider:
    endpoint: db-prod-my-app-db.abc123.us-east-1.rds.amazonaws.com
```

### MR 2: Security Group
```yaml
apiVersion: ec2.aws.upbound.io/v1beta1
kind: SecurityGroup
metadata:
  name: db-prod-my-app-db-sg
  ownerReferences: [...]
spec:
  forProvider:
    region: us-east-1
    description: Security group for database
    # ... configuration
```

**Explanation**:
- Created automatically by Composition
- Have ownerReferences pointing to parent XR
- Represent actual cloud resources
- Status reflects real cloud state

---

## Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│ 1. USER CREATES CLAIM                                       │
│    kubectl apply -f database-claim.yaml                     │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. CROSSPLANE CREATES XR (Composite Resource)               │
│    Based on XRD definition                                  │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. COMPOSITION SELECTED                                     │
│    Matches compositionSelector labels                       │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. PIPELINE EXECUTION                                       │
│    ┌──────────────────────────────────────────────┐         │
│    │ Step 1: function-go-templating               │         │
│    │ - Processes input parameters                 │         │
│    │ - Calculates instanceSize, dbName            │         │
│    │ - Creates context                            │         │
│    └──────────────┬───────────────────────────────┘         │
│                   ▼                                          │
│    ┌──────────────────────────────────────────────┐         │
│    │ Step 2: function-go-templating               │         │
│    │ - Renders MRs using context                  │         │
│    │ - Creates RDS, SecurityGroup, Secret         │         │
│    └──────────────┬───────────────────────────────┘         │
│                   ▼                                          │
│    ┌──────────────────────────────────────────────┐         │
│    │ Step 3: function-auto-ready                  │         │
│    │ - Marks resources ready when conditions met  │         │
│    └──────────────────────────────────────────────┘         │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. MANAGED RESOURCES CREATED                                │
│    - RDS Instance (db-prod-my-app-db)                       │
│    - Security Group (db-prod-my-app-db-sg)                  │
│    - Kubernetes Secret (db-prod-my-app-db-connection)       │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. CLOUD PROVIDER PROVISIONS                                │
│    AWS creates actual RDS database                          │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 7. STATUS UPDATED                                           │
│    XR and Claim show Ready=True                             │
│    Connection details available in Secret                   │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Concepts Explained

### 1. **XRD (CompositeResourceDefinition)**
- Like a CRD for your platform
- Defines the contract (API) between users and platform
- Specifies inputs (spec) and outputs (status)

### 2. **Composition**
- The "how" - implementation details
- Translates XR into cloud resources
- Can have multiple compositions for same XRD (e.g., AWS vs Azure)

### 3. **Functions**
- Plugins in the pipeline
- Transform, validate, or generate resources
- Can be Go templates, Python, or custom code

### 4. **Claim (CR)**
- User-facing API
- Namespace-scoped (for multi-tenancy)
- Simple, high-level parameters

### 5. **XR (Composite Resource)**
- Cluster-scoped
- Created automatically from Claim
- Owns all Managed Resources

### 6. **MR (Managed Resource)**
- Represents real cloud resource
- Has 1:1 mapping to cloud API
- Automatically reconciled with cloud state

---

## Apply in Order

```bash
# 1. Install required providers
kubectl apply -f - <<EOF
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/upbound/provider-aws:v0.40.0
---
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-go-templating
spec:
  package: xpkg.upbound.io/crossplane-contrib/function-go-templating:v0.4.0
---
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-auto-ready
spec:
  package: xpkg.upbound.io/crossplane-contrib/function-auto-ready:v0.2.0
EOF

# 2. Apply XRD
kubectl apply -f definition.yaml

# 3. Apply Composition
kubectl apply -f composition.yaml

# 4. Create database (user action)
kubectl apply -f database-claim.yaml

# 5. Check status
kubectl get database -n production
kubectl describe database my-app-db -n production

# 6. View created resources
kubectl get managed
kubectl get xdatabase
```

---

## Checking Results

```bash
# View the claim
kubectl get database my-app-db -n production -o yaml

# View the composite resource
kubectl get xdatabase -o yaml

# View managed resources
kubectl get rds
kubectl get securitygroup
kubectl get object

# Get connection details
kubectl get secret my-app-db-connection -n production -o yaml
```

---

## Summary

| Component | Type | Scope | Created By | Purpose |
|-----------|------|-------|------------|---------|
| XRD | Platform | Cluster | Platform Team | Define API schema |
| Composition | Platform | Cluster | Platform Team | Define implementation |
| Function | Platform | Cluster | Crossplane | Process/transform resources |
| Claim (CR) | User | Namespace | App Team | Request infrastructure |
| XR | Internal | Cluster | Crossplane | Manage lifecycle |
| MR | Internal | Cluster | Crossplane | Represent cloud resources |

This architecture provides:
- **Separation of concerns**: Platform team builds, app team consumes
- **Abstraction**: Hide cloud complexity
- **Consistency**: Same API across environments
- **Self-service**: Developers provision without tickets
