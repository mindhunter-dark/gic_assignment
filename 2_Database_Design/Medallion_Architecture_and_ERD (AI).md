# Medallion Data Architecture

## 1. Medallion Architecture – Data Flow

```mermaid
%%{init: {
  "flowchart": {
    "htmlLabels": true,
    "nodeSpacing": 45,
    "rankSpacing": 70,
    "wrappingWidth": 500
  }
}}%%

flowchart TB

    subgraph BR["BRONZE LAYER — Raw Data"]
        direction LR
		
		BRNOTE["<div style='text-align:left;width:420px'><b style='font-size:20'>Ingestion Operations</b><br/>• Append-only ingestion; never overwrite source history<br/>• Preserve source values without business transformations<br/>• Add record_created_on and record_source metadata</div>"]

        BC["<div style='text-align:left;width:310px'><b style='font-size:20'>course</b><br/>course_id<br/>name<br/>description<br/>prerequisites<br/>record_created_on<br/>record_source</div>"]

        BE["<div style='text-align:left;width:310px'><b style='font-size:20'>enrollment</b><br/>enrollment_id<br/>participant_id<br/>participant_name<br/>course_id<br/>course_date<br/>amount<br/>subsidy<br/>credits_used<br/>record_created_on<br/>record_source</div>"]

        BC ~~~ BE
        BE ~~~ BRNOTE
    end

    subgraph SILVER_GROUP["SILVER LAYER — Cleaned, Validated and Normalized Data"]
        direction TB

        SITR["<div style='text-align:left;width:650px;height:200px;'><b style='font-size:20'>Transformations &amp; Processing</b><br/>• Deduplicate using ROW_NUMBER() and select the latest record by record_created_on<br/>• Validate source fields and apply data-quality rules<br/>• Cast identifiers to integers and financial values to floats<br/>• Split course prerequisites into individual mapping records<br/>• Parse course_date and look up course_date_id in ref_date<br/>• Calculate amount_paid_by_enrollee = amount - (subsidy + credits_used)<br/>• Retain Bronze ingestion metadata for traceability</div>"]

        subgraph SILVER_CONTENT[" "]
            direction LR

            subgraph ANALYTICAL["Analytical Tables"]
                direction TB

                subgraph DIMENSIONS[" "]
                    direction LR
                    DC["<div style='text-align:left;width:260px'><b style='font-size:20'>dim_course</b><br/><u>PK</u> course_id<br/>course_name<br/>description<br/>record_created_on<br/>record_created_on_bronze<br/>record_source</div>"]

                    DP["<div style='text-align:left;width:260px'><b style='font-size:20'>dim_participant</b><br/><u>PK</u> participant_id<br/>participant_name<br/>record_created_on<br/>record_created_on_bronze<br/>record_source</div>"]
                end

                subgraph OTHER_TABLES[" "]
                    direction LR
                    MP["<div style='text-align:left;width:290px'><b style='font-size:20'>map_prerequisite</b><br/><u>PK</u> prerequisite_id<br/><u>FK</u> course_id<br/><u>FK</u> prerequisite_course_id<br/>record_created_on<br/>record_created_on_bronze<br/>record_source</div>"]

                    RD["<div style='text-align:left;width:230px'><b style='font-size:20'>ref_date</b><br/><u>PK</u> date_id<br/>date</div>"]
                end

                FE["<div style='text-align:left;width:310px'><b style='font-size:20'>fact_enrollment</b><br/><u>PK</u> enrollment_id<br/><u>FK</u> participant_id<br/><u>FK</u> course_id<br/><u>FK</u> course_date_id<br/>amount<br/>subsidy<br/>credits_used<br/>amount_paid_by_enrollee<br/>record_created_on<br/>record_created_on_bronze<br/>record_source</div>"]

                DC --> FE
                DP --> FE
                MP ~~~ RD
            end

            subgraph QU["Quarantine Tables"]
                direction TB

                QOPS["<div style='text-align:left;width:430px;height:175px;'><b style='font-size:20'>Quarantine Operations</b><br/>• Store records that fail quarantine-level checks<br/>• Retain original source fields and failed_check<br/>• Create one quarantine record per failed check<br/>• Alert the source for rectification<br/>• Do not propagate quarantined records to analytical tables</div>"]

                QC["<div style='text-align:left;width:260px'><b style='font-size:20'>qua_course</b><br/>Source course fields<br/>failed_check<br/>record_created_on<br/>record_created_on_bronze<br/>record_source</div>"]

                QE["<div style='text-align:left;width:300px'><b style='font-size:20'>qua_enrollment</b><br/>Source enrollment fields<br/>failed_check<br/>record_created_on<br/>record_created_on_bronze<br/>record_source</div>"]

                QOPS --> QC
                QOPS --> QE
            end
        end

        SITR --> DC
        SITR --> DP
        SITR --> MP
        SITR --> RD
        SITR --> FE
        SITR --> QOPS
    end

    subgraph GOLD["GOLD LAYER — Business-ready Analytics"]
        direction LR

        GM["<div style='text-align:left;width:340px'><b style='font-size:20'>monthly_performance</b><br/>course_joined_year<br/>course_joined_month<br/>total_amount<br/>subsidy<br/>credits_used<br/>amount_paid_by_enrollee<br/>no_of_enrollments<br/>no_of_unique_enrollments<br/>no_of_courses<br/>no_of_participants</div>"]

        GC["<div style='text-align:left;width:340px'><b style='font-size:20'>course_360</b><br/>course_joined_year<br/>course_joined_month<br/>course_id<br/>course_name<br/>course_description<br/>total_amount<br/>subsidy<br/>credits_used<br/>amount_paid_by_enrollee<br/>no_of_enrollments<br/>no_of_unique_enrollments<br/>no_of_participants</div>"]

        GP["<div style='text-align:left;width:340px'><b style='font-size:20'>participant_360</b><br/>course_joined_year<br/>participant_id<br/>participant_name<br/>total_amount<br/>subsidy<br/>credits_used<br/>amount_paid_by_enrollee<br/>no_of_enrollments<br/>no_of_unique_enrollments<br/>no_of_courses</div>"]

        GOT["<div style='text-align:left;width:650px'><b style='font-size:20'>Gold Transformations</b><br/>• Join fact_enrollment to ref_date to derive year and month<br/>• Join dimensions where descriptive attributes are required<br/>• Optionally exclude unknown courses (course_id = -1)<br/>• Aggregate financial measures and enrollment/participant metrics<br/>• Build pre-aggregated, denormalized tables for business use<br/>• Index frequently queried columns and partition by year/month where appropriate</div>"]

        GM ~~~ GC
        GC ~~~ GP
        GM ~~~ GOT
        GC ~~~ GOT
        GP ~~~ GOT
    end

    %% Bronze to Silver data flow
    BC --> SITR
    BE --> SITR

    %% Silver to Gold data flow
    FE --> GM
    FE --> GC
    FE --> GP

    %% Layer styling
    classDef bronze fill:#e8f1fc,stroke:#315b85,stroke-width:1.5px,color:#172b4d;
    classDef silver fill:#e6f4ea,stroke:#34734b,stroke-width:1.5px,color:#173d25;
    classDef gold fill:#fff1dc,stroke:#a66a20,stroke-width:1.5px,color:#57370c;
    classDef quarantine fill:#fde8e7,stroke:#a33b36,stroke-width:1.5px,color:#5b1d1a;
    classDef process fill:#f1f5f9,stroke:#64748b,stroke-width:1.5px,color:#1e293b;

    class BC,BE bronze;
    class BRNOTE,SITR process;
    class DC,DP,MP,RD,FE silver;
    class QOPS,QC,QE quarantine;
    class GM,GC,GP gold;

    style BR fill:#f4f8fd,stroke:#315b85,stroke-width:2px
    style SILVER_GROUP fill:#f3faf5,stroke:#34734b,stroke-width:2px
    style SILVER_CONTENT fill:#f3faf5,stroke:#34734b,stroke-width:1px
    style ANALYTICAL fill:#f8fcf9,stroke:#8ab69a,stroke-width:1px
    style DIMENSIONS fill:#f8fcf9,stroke:#8ab69a,stroke-width:1px
    style OTHER_TABLES fill:#f8fcf9,stroke:#8ab69a,stroke-width:1px
    style QU fill:#fff7f6,stroke:#a33b36,stroke-width:1.5px
    style GOLD fill:#fff8ed,stroke:#a66a20,stroke-width:2px
```

## 2. Data Quality Checks

These checks are performed between the Bronze and Silver layers. Depending on the check, failed records are either quarantined or allowed to continue into Silver with an appropriate alert.

| Source table | Field | Data quality check | Action on failure |
|---|---|---|---|
| course | course_id | Must be a non-null integer | Quarantine and alert |
| course | prerequisites | Must be a valid comma-separated array of existing course IDs | Alert; continue ingestion |
| enrollment | enrollment_id | Must be a non-null integer | Quarantine and alert |
| enrollment | participant_id | Must be a non-null integer | Quarantine and alert |
| enrollment | course_id | Must be a non-null integer present in dim_course | Alert; use placeholder course_id = -1 and course_name = 'Unknown' |
| enrollment | course_date | Must be parseable into a valid date | Quarantine and alert |
| enrollment | amount | Must be parseable into a decimal value | Quarantine and alert |
| enrollment | subsidy | Must be parseable into a decimal value OR NULL | Quarantine and alert if not parseable; allow NULL |
| enrollment | credits_used | Must be parseable into a decimal value OR NULL | Quarantine and alert if not parseable; allow NULL |

### Quarantine Tables

Quarantine tables are maintained in the Silver layer.

| Table | Contents | Additional field |
|---|---|---|
| qua_course | Failed course records, retaining original source fields | failed_check: NVARCHAR |
| qua_enrollment | Failed enrollment records, retaining original source fields | failed_check: NVARCHAR |

**Quarantine processing rules:**
- Only records that fail quarantine-designated checks are stored.
- Original source values are retained.
- The `failed_check` field records the failed check and its reason.
- A single Bronze record can generate multiple quarantine records if it fails multiple checks.
- Quarantined records do not continue into Silver's analytical tables.
- Alerts should be sent to the source for rectification.
- NULL values for `subsidy` and `credits_used` are permitted and do not result in quarantine.

---

## 3. Entity Relationship Diagram (ERD)

The following ER diagram illustrates the relationships across all three layers.

```mermaid
%%{init: {"theme":"base","themeVariables":{"background":"#ffffff","primaryColor":"#e8f1fc","primaryTextColor":"#1e293b","primaryBorderColor":"#315b85","lineColor":"#64748b","fontFamily":"Arial","er":{"fontSize":"12px","entityPadding":12}}}}%%
erDiagram

    BRONZE_COURSE {
        nvarchar course_id
        nvarchar name
        nvarchar description
        nvarchar prerequisites
        timestamp record_created_on
        nvarchar record_source
    }

    BRONZE_ENROLLMENT {
        nvarchar enrollment_id
        nvarchar participant_id
        nvarchar participant_name
        nvarchar course_id
        nvarchar course_date
        nvarchar amount
        nvarchar subsidy
        nvarchar credits_used
        timestamp record_created_on
        nvarchar record_source
    }

    SILVER_DIM_COURSE {
        int course_id PK
        nvarchar course_name
        nvarchar description
        timestamp record_created_on
        timestamp record_created_on_bronze
        nvarchar record_source
    }

    SILVER_DIM_PARTICIPANT {
        int participant_id PK
        nvarchar participant_name
        timestamp record_created_on
        timestamp record_created_on_bronze
        nvarchar record_source
    }

    SILVER_MAP_PREREQUISITE {
        int prerequisite_id PK
        int course_id FK
        int prerequisite_course_id FK
        timestamp record_created_on
        timestamp record_created_on_bronze
        nvarchar record_source
    }

    SILVER_REF_DATE {
        int date_id PK
        date date
    }

    SILVER_FACT_ENROLLMENT {
        int enrollment_id PK
        int participant_id FK
        int course_id FK
        int course_date_id FK
        float amount
        float subsidy
        float credits_used
        float amount_paid_by_enrollee
        timestamp record_created_on
        timestamp record_created_on_bronze
        nvarchar record_source
    }

    SILVER_QUA_COURSE {
        nvarchar course_id
        nvarchar name
        nvarchar description
        nvarchar prerequisites
        nvarchar failed_check
        timestamp record_created_on
        timestamp record_created_on_bronze
        nvarchar record_source
    }

    SILVER_QUA_ENROLLMENT {
        nvarchar enrollment_id
        nvarchar participant_id
        nvarchar participant_name
        nvarchar course_id
        nvarchar course_date
        nvarchar amount
        nvarchar subsidy
        nvarchar credits_used
        nvarchar failed_check
        timestamp record_created_on
        timestamp record_created_on_bronze
        nvarchar record_source
    }

    GOLD_MONTHLY_PERFORMANCE {
        int course_joined_year
        string course_joined_month
        float total_amount
        float subsidy
        float credits_used
        float amount_paid_by_enrollee
        int no_of_enrollments
        int no_of_unique_enrollments
        int no_of_courses
        int no_of_participants
    }

    GOLD_COURSE_360 {
        int course_joined_year
        string course_joined_month
        int course_id
        nvarchar course_name
        nvarchar course_description
        float total_amount
        float subsidy
        float credits_used
        float amount_paid_by_enrollee
        int no_of_enrollments
        int no_of_unique_enrollments
        int no_of_participants
    }

    GOLD_PARTICIPANT_360 {
        int course_joined_year
        int participant_id
        nvarchar participant_name
        float total_amount
        float subsidy
        float credits_used
        float amount_paid_by_enrollee
        int no_of_enrollments
        int no_of_unique_enrollments
        int no_of_courses
    }

    %% Bronze relationships
    BRONZE_COURSE o|--o{ BRONZE_ENROLLMENT : "course_id"

    %% Bronze to Silver lineage
    BRONZE_COURSE ||--o{ SILVER_DIM_COURSE : "deduplicate and cast"
    BRONZE_ENROLLMENT ||--o{ SILVER_DIM_PARTICIPANT : "deduplicate and cast"
    BRONZE_ENROLLMENT ||--o{ SILVER_FACT_ENROLLMENT : "cast and transform"

    %% Silver dimension and fact relationships
    SILVER_DIM_PARTICIPANT ||--o{ SILVER_FACT_ENROLLMENT : "participant_id"
    SILVER_DIM_COURSE ||--o{ SILVER_FACT_ENROLLMENT : "course_id"
    SILVER_REF_DATE ||--o{ SILVER_FACT_ENROLLMENT : "course_date_id"

    %% Self-referencing prerequisite mapping
    SILVER_DIM_COURSE ||--o{ SILVER_MAP_PREREQUISITE : "course_id"
    SILVER_DIM_COURSE ||--o{ SILVER_MAP_PREREQUISITE : "prerequisite_course_id"

    %% Silver to Gold lineage
    SILVER_FACT_ENROLLMENT ||--o{ GOLD_MONTHLY_PERFORMANCE : "aggregate"
    SILVER_FACT_ENROLLMENT ||--o{ GOLD_COURSE_360 : "aggregate"
    SILVER_FACT_ENROLLMENT ||--o{ GOLD_PARTICIPANT_360 : "aggregate"

    SILVER_REF_DATE ||--o{ GOLD_MONTHLY_PERFORMANCE : "derive year and month"
    SILVER_REF_DATE ||--o{ GOLD_COURSE_360 : "derive year and month"
    SILVER_REF_DATE ||--o{ GOLD_PARTICIPANT_360 : "derive year"

    SILVER_DIM_COURSE ||--o{ GOLD_COURSE_360 : "course attributes"
    SILVER_DIM_PARTICIPANT ||--o{ GOLD_PARTICIPANT_360 : "participant attributes"
```

### ERD Relationship Notes

| Relationship | Cardinality | Explanation |
|---|---|---|
| dim_course → fact_enrollment | 1:N | A course can have multiple enrollments |
| dim_participant → fact_enrollment | 1:N | A participant can have multiple enrollments |
| ref_date → fact_enrollment | 1:N | A date can be associated with multiple enrollments |
| dim_course (course_id) → map_prerequisite | 1:N | A course can have multiple prerequisite mappings as the course being studied |
| dim_course (prerequisite_course_id) → map_prerequisite | 1:N | A course can be a prerequisite for multiple other courses |

**Important modelling considerations:**

1. Bronze relationships represent logical source relationships. The source tables are not explicitly defined with primary and foreign key constraints.
2. The Silver fact table has a primary key on `enrollment_id`, reflecting the assumption that one enrollment ID maps to exactly one course.
3. The prerequisite mapping table has two foreign keys pointing to `dim_course`, representing the course and its prerequisite.
4. Quarantine tables are exception-storage tables. They are not part of the analytical relational model and therefore have no declared foreign key relationships.
5. Gold tables are pre-aggregated analytical outputs. Their schemas do not specify primary keys or foreign keys; the ERD therefore shows their data lineage rather than asserting physical constraints.

---

## 4. Gold Layer – Aggregation Logic

| Gold table | Grouping dimensions | Key operations |
|---|---|---|
| monthly_performance | course_joined_year, course_joined_month | SUM financial measures; COUNT DISTINCT enrollment, enrollment + participant, courses and participants |
| course_360 | course_joined_year, course_joined_month, course_id, course_name, course_description | SUM financial measures; COUNT DISTINCT enrollment, enrollment + participant and participants |
| participant_360 | course_joined_year, participant_id, participant_name | SUM financial measures; COUNT DISTINCT enrollment, enrollment + participant and courses |

### Performance Considerations

- Index frequently queried columns in Silver and Gold.
- Partition Gold tables using `course_joined_year` and `course_joined_month` where appropriate.
- Maintain the `ref_date` table by adding future dates periodically.
- Allow Gold tables to evolve according to business reporting requirements.
- Consider filtering `course_id = -1` when reporting requires only records associated with known courses.

## 5. Notes

- This document, until '4. Gold Layer – Aggregation Logic', was generated by AI using the input from the document 'Database_Design_Document.md', created by me, and some prompting. I've modified some HTML code in '1. Medallion Architecture – Data Flow' and briefly reviewed the document for any inconsistencies.
- Click this [link](https://chatgpt.com/share/6af0a786-6868-83ec-b1d7-c0038305c62d) for the conversation that I had with ChatGPT to create this document.