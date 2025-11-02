# YAML Fundamentals

## What is YAML?

**YAML** stands for "YAML Ain't Markup Language" and is a human-readable data serialization language. It's commonly used for:
- Configuration files (Kubernetes, Docker Compose, Ansible)
- Data exchange between systems
- Application settings

**Key characteristics**:
- Human-readable and easy to understand
- Uses indentation (whitespace) to show structure
- Supports multiple data types (strings, numbers, lists, dictionaries, etc.)
- Language-agnostic (works with any programming language)

---

## YAML Basic Syntax Rules

### 1. Key-Value Pairs (Dictionaries/Maps)

A key-value pair is the fundamental building block of YAML.

**Syntax**:
```yaml
key: value
```

**Examples**:
```yaml
name: John
age: 30
country: USA
is_student: false
```

**In other languages**:
- Python: `{"name": "John", "age": 30}`
- JSON: `{"name": "John", "age": 30}`

### 2. Indentation (Whitespace Matters!)

YAML uses **indentation** to show nesting and hierarchy. **Always use spaces, never tabs!**

**Example of nested structure**:
```yaml
person:
  name: John
  age: 30
  address:
    city: New York
    country: USA
```

**Important rules**:
- Use **consistent spacing** (typically 2 or 4 spaces)
- Each indentation level increases by the same amount
- Indentation determines parent-child relationships

**❌ Bad (inconsistent indentation)**:
```yaml
person:
  name: John
    age: 30  # Wrong! Inconsistent spacing
```

**✅ Good (consistent indentation)**:
```yaml
person:
  name: John
  age: 30
```

### 3. Lists/Arrays

Lists are ordered collections of items, indicated by a dash `-` at the start of each item.

**Syntax**:
```yaml
- item1
- item2
- item3
```

**Example - List of strings**:
```yaml
fruits:
  - apple
  - banana
  - orange
  - grape
```

**Example - List of objects (dictionaries)**:
```yaml
employees:
  - name: John
    age: 30
    role: Engineer
  - name: Sarah
    age: 28
    role: Designer
  - name: Mike
    age: 35
    role: Manager
```

**In Python**:
```python
employees = [
  {"name": "John", "age": 30, "role": "Engineer"},
  {"name": "Sarah", "age": 28, "role": "Designer"},
  {"name": "Mike", "age": 35, "role": "Manager"}
]
```

### 4. Nested Structures

Combine dictionaries and lists to create complex nested structures.

**Example**:
```yaml
company:
  name: TechCorp
  departments:
    - name: Engineering
      employees:
        - John
        - Sarah
      budget: 500000
    - name: Marketing
      employees:
        - Alice
        - Bob
      budget: 200000
```

---

## Data Types in YAML

### String
```yaml
name: "John Doe"           # Quoted string
city: New York             # Unquoted string
description: |             # Multi-line string (preserves newlines)
  This is a long
  multi-line string
```

### Number (Integer and Float)
```yaml
age: 30                    # Integer
height: 5.9                # Float
price: 19.99
count: -5                  # Negative number
```

### Boolean
```yaml
is_active: true            # Boolean true
is_deleted: false          # Boolean false
is_student: yes            # 'yes' is also true
is_admin: no               # 'no' is also false
```

### Null
```yaml
middle_name: null          # Null value
description:               # Empty = null
```

### Date/Time
```yaml
birth_date: 1990-01-15     # Date (ISO 8601 format)
timestamp: 2024-10-29T10:30:00Z  # DateTime
```

---

## Common YAML Patterns in Kubernetes

### Kubernetes Pod Definition

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  labels:
    app: web
    version: v1
spec:
  containers:
  - name: web-container
    image: nginx:latest
    ports:
    - containerPort: 80
      name: http
    - containerPort: 443
      name: https
    env:
    - name: LOG_LEVEL
      value: "DEBUG"
    - name: DATABASE_URL
      value: "postgresql://localhost:5432"
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web
        image: myapp:v1
        ports:
        - containerPort: 8080
```

### Kubernetes Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
    nodePort: 30080
```

---

## YAML vs JSON

Both YAML and JSON represent the same data, but with different syntax:

**Same data in JSON**:
```json
{
  "name": "John",
  "age": 30,
  "skills": ["Python", "JavaScript", "Go"],
  "address": {
    "city": "New York",
    "country": "USA"
  }
}
```

**Same data in YAML**:
```yaml
name: John
age: 30
skills:
  - Python
  - JavaScript
  - Go
address:
  city: New York
  country: USA
```

**YAML advantages**:
- More readable (less punctuation)
- Easier to edit manually
- Better for configuration files
- Smaller file size

---

## Common YAML Mistakes

### ❌ Mixing Tabs and Spaces
```yaml
person:
	name: John    # Wrong! Tab used instead of spaces
  age: 30
```
**Fix**: Use spaces consistently (usually 2 or 4)

### ❌ Inconsistent Indentation
```yaml
person:
  name: John
    age: 30   # Wrong! Indentation increased suddenly
  city: New York
```
**Fix**: Keep indentation consistent within the same level

### ❌ Forgetting Indentation for Nested Items
```yaml
person:
name: John     # Wrong! Should be indented
age: 30
```
**Fix**: Indent child items under their parent

### ❌ Using Quotes Incorrectly
```yaml
password: "123:456"  # This is fine
message: "This is "quoted" text"  # Wrong! Nested quotes cause issues
```
**Fix**: Use single quotes for strings containing double quotes: `'This is "quoted" text'`

---

## Practical Examples

### Example 1: Configuration File

```yaml
# app-config.yaml
application:
  name: MyApp
  version: 1.0.0
  debug: true

database:
  host: localhost
  port: 5432
  username: admin
  password: secret123

logging:
  level: INFO
  output: stdout

features:
  - authentication
  - api_gateway
  - caching
```

### Example 2: Complex Nested Structure

```yaml
# team.yaml
company: TechCorp
teams:
  - team_name: Backend
    manager: John Smith
    members:
      - name: Alice
        role: Senior Engineer
        skills:
          - Python
          - Go
          - Kubernetes
      - name: Bob
        role: Junior Engineer
        skills:
          - JavaScript
          - Node.js

  - team_name: Frontend
    manager: Sarah Jones
    members:
      - name: Charlie
        role: Senior Designer
        skills:
          - React
          - CSS
          - UX Design
```

### Example 3: Kubernetes Resource with All Features

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
  labels:
    app: myapp
    version: v1
data:
  database_url: "postgresql://db:5432/myapp"
  log_level: "INFO"
  features: |
    - feature_a
    - feature_b
    - feature_c
```

---

## Key Takeaways

1. **YAML uses indentation**: Hierarchy is shown by consistent spacing (spaces, not tabs)
2. **Key-value pairs**: Basic building blocks (`key: value`)
3. **Lists with dashes**: Items in arrays start with `-`
4. **Dictionaries (objects)**: Use indentation to nest related data
5. **Multiple data types**: Strings, numbers, booleans, null, dates all supported
6. **Kubernetes uses YAML**: All resource definitions (Pods, Deployments, Services) are YAML
7. **Readability matters**: YAML's strength is being human-readable
8. **Consistency is critical**: Use the same spacing throughout (typically 2 or 4 spaces per level)
