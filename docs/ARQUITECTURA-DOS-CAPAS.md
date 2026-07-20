# Arquitectura de dos capas: IA en cómo construimos + IA en lo que entregamos

## Diagrama 1 — Vista ejecutiva (para el cliente)

```mermaid
flowchart TB
    subgraph BUILD["🛠️ CAPA 1 · IA EN CÓMO CONSTRUIMOS"]
        direction TB
        DEV["Equipo de desarrollo"]
        GUARD["ai_code_guardrails<br/>Skills · Agentes · Rules · Hooks"]
        GATE{"Controles de calidad<br/>PHPCS · PHPUnit · Revisión"}
        DEV --> GUARD --> GATE
    end

    subgraph RUN["🚀 CAPA 2 · IA EN LO QUE ENTREGAMOS"]
        direction TB
        USER["Editores y usuarios finales"]
        DRUPAL["Sitio Drupal 11"]
        AI["Capa de abstracción de IA<br/>drupal/ai"]
        PROV["Proveedores intercambiables<br/>Anthropic · OpenAI · Ollama local"]
        HUMAN{"Revisión humana<br/>antes de publicar"}
        USER --> DRUPAL --> AI --> PROV
        AI --> HUMAN
    end

    GATE -->|"código validado se despliega"| DRUPAL

    style BUILD fill:#e8f0fe,stroke:#1a73e8,stroke-width:2px
    style RUN fill:#e6f4ea,stroke:#137333,stroke-width:2px
    style HUMAN fill:#fef7e0,stroke:#f9ab00,stroke-width:2px
    style GATE fill:#fef7e0,stroke:#f9ab00,stroke-width:2px
```

**Mensaje:** la misma disciplina de ingeniería se aplica en las dos capas. No es "le pusimos un chatbot al sitio".

---

## Diagrama 2 — Vista técnica (para el equipo del cliente)

```mermaid
flowchart LR
    subgraph L1["CAPA 1 · Build time · corre en la máquina del dev"]
        direction TB
        D1["Desarrollador"]
        D2["Claude Code"]
        D3["Skills<br/>seguridad, DI, plugins,<br/>caching, migraciones"]
        D4["Agentes<br/>backend-dev, reviewer,<br/>quality-gate, done-gate"]
        D5["Rules<br/>convenciones siempre activas"]
        D6["Hooks<br/>contexto de sesión"]
        D7["Git · GitHub CLI · DDEV · Drush"]
        D1 --> D2
        D3 --> D2
        D4 --> D2
        D5 --> D2
        D6 --> D2
        D2 --> D7
    end

    subgraph L2["CAPA 2 · Runtime · corre dentro del sitio"]
        direction TB
        R1["Editor / Visitante"]
        R2["Drupal 11 · Olivero"]
        R3["drupal/ai<br/>abstracción de proveedor"]
        R4["CKEditor AI<br/>asistencia editorial"]
        R5["AI Agents<br/>tool-calling de solo lectura"]
        R6["AI Search<br/>Search API + vectores"]
        R7["Contenido<br/>Artículos · Cursos · Expertos"]
        R8["Key module<br/>credenciales fuera de Git"]
        R9{"Revisión humana"}
        P1["Anthropic"]
        P2["OpenAI"]
        P3["Ollama local"]

        R1 --> R2
        R2 --> R4 --> R3
        R2 --> R5 --> R3
        R2 --> R6 --> R7
        R5 --> R7
        R3 --> P1
        R3 --> P2
        R3 --> P3
        R8 -.credenciales.-> R3
        R4 --> R9
        R5 --> R9
        R9 --> R7
    end

    D7 ==>|"deploy"| R2

    style L1 fill:#e8f0fe,stroke:#1a73e8
    style L2 fill:#e6f4ea,stroke:#137333
    style R9 fill:#fef7e0,stroke:#f9ab00,stroke-width:2px
    style R8 fill:#fce8e6,stroke:#c5221f
```

---

## Diagrama 3 — El flujo de una petición con gobernanza

```mermaid
sequenceDiagram
    actor E as Editor
    participant D as Drupal
    participant K as Key module
    participant A as drupal/ai
    participant P as Proveedor
    participant R as Revisión

    E->>D: "Propón un resumen de este artículo"
    D->>D: ¿El rol tiene permiso de IA?
    D->>K: Solicita credencial
    K-->>A: Credencial desde variable de entorno
    D->>A: Petición + contexto del contenido
    A->>P: Llamada al modelo seleccionado
    P-->>A: Respuesta generada
    A-->>D: Propuesta (NO se guarda)
    D->>E: Muestra propuesta
    E->>R: Acepta, edita o descarta
    R->>D: Solo entonces se guarda
    Note over R,D: Nada generado por IA<br/>se publica sin humano
```

---

## Notas para presentar

- **Capa 1 no se despliega.** Vive en el entorno de desarrollo. Cero superficie de ataque en producción, cero dependencias nuevas en el sitio.
- **Capa 2 sí se despliega.** Por eso lleva Key module, permisos por rol, logging y revisión humana obligatoria.
- **El rombo amarillo es el punto de venta.** La IA propone, la persona decide. Nunca al revés.
- **Los tres proveedores en paralelo** son el argumento anti-lock-in: si el cliente quiere todo local por privacidad, Ollama; si quiere máxima calidad, Anthropic. Mismo código.
