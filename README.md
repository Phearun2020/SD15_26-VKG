# HR-SMIS Knowledge Graph Demo

This project demonstrates integration between HR data and SMIS data through one shared person entity identified by a generated `canonical_id`.

## Flow

1. Dremio federates HR and SMIS data.
2. Morph-KGC queries Dremio directly.
3. The mapping creates one shared person resource:

```ttl
http://smis.itc.edu.kh/data/person/{canonical_id}
```

4. HR and SMIS attributes are attached to that same subject.
5. The shared person is linked to SMIS department and branch resources.

## Current RDF Model

The main mapping is in [mapping/mapping.ttl](mapping/mapping.ttl).

The shared person resource contains integrated attributes from both systems, including:

```ttl
ex:canonicalId
ex:hrId
ex:hrEmail
ex:smisId
ex:employeeId
ex:email
ex:name
ex:phoneNumber
```

The subject is typed as:

```ttl
ex:HRPerson
ex:Employee
```

Department and branch stay as separate SMIS resources:

```ttl
http://smis.itc.edu.kh/data/department/{department_id}
http://smis.itc.edu.kh/data/branch/{branch_id}
```

And the links are:

```ttl
ont:hasDepartment
ont:hasBranch
```

## Minimal Demonstrator

The minimal demonstrator should show:

1. data coming from different systems
2. integration through a common canonical identifier

In this project:

- `hr_id` comes from the HR source
- `phoneNumber` comes from the SMIS source
- both are attached to the same `canonical_id`

Example Dremio SQL:

```sql
SELECT DISTINCT
  CONCAT(
    'P_',
    SPLIT_PART(LOWER(h.email), '@', 1),
    '_',
    SPLIT_PART(LOWER(h.email), '@', 2)
  ) AS canonical_id,
  h.salary_id AS hr_id,
  e.phone AS phone_number
FROM "@phearun".hr h
INNER JOIN smis.public.users s
  ON LOWER(h.email) = LOWER(s.email)
INNER JOIN smis.public.employees e
  ON s.id = e.user_id
WHERE h.salary_id IS NOT NULL
  AND e.phone IS NOT NULL
LIMIT 20
```

Example SPARQL:

```sparql
PREFIX ex: <http://smis.itc.edu.kh/ont#>

SELECT ?canonicalId ?hrId ?phone WHERE {
  ?employee a ex:HRPerson ;
            ex:canonicalId ?canonicalId ;
            ex:hrId ?hrId ;
            ex:phoneNumber ?phone .
}
LIMIT 5
```

## Dremio Connection

Morph-KGC connects directly to Dremio through [config.ini](config.ini):

```ini
[DataSource1]
mappings: mapping/mapping.ttl
db_url: dremio+flight://{DREMIO_USER}:{DREMIO_PASSWORD_URLENCODED}@{DREMIO_HOST}:{DREMIO_FLIGHT_PORT}/dremio?UseEncryption=false
```

Set the required environment variables before running:

```bash
export DREMIO_USER="your_username"
export DREMIO_PASSWORD="your_password"
export DREMIO_PASSWORD_URLENCODED="$(python -c 'from urllib.parse import quote_plus; import os; print(quote_plus(os.environ["DREMIO_PASSWORD"]))')"
export DREMIO_HOST="localhost"
export DREMIO_FLIGHT_PORT="32010"
```

`DREMIO_PASSWORD_URLENCODED` is needed because special characters such as `@` must be URL-encoded for the SQLAlchemy connection string.

## Main Mapping Structure

The mapping currently has these main parts:

- `SharedPersonMap`
- `SharedPersonDepartmentLinkMap`
- `SharedPersonBranchLinkMap`
- `DepartmentMap`
- `BranchMap`

`SharedPersonMap` performs the HR-SMIS integration with a join on email:

```sql
FROM "@phearun".hr h
INNER JOIN smis.public.users s
  ON LOWER(h.email) = LOWER(s.email)
INNER JOIN smis.public.employees e
  ON s.id = e.user_id
```

The `canonical_id` is generated as:

```sql
CONCAT(
  'P_',
  SPLIT_PART(LOWER(h.email), '@', 1),
  '_',
  SPLIT_PART(LOWER(h.email), '@', 2)
)
```

## Run

From the project root:

```bash
source .venv/bin/activate
python -m morph_kgc config.ini
```

This writes the RDF output to:

```text
output/output.ttl
```

## Dependencies

Install the required packages:

```bash
pip install rdflib morph_kgc pandas sqlalchemy sqlalchemy_dremio
```

`sqlalchemy_dremio` is required because Morph-KGC uses SQLAlchemy for relational database sources.

## Notes

- Dremio must be running before Morph-KGC runs.
- The Dremio Flight port is usually `32010`.
- The Dremio web UI is usually available at `http://localhost:9047`.
- If the HR table name changes in Dremio, update the SQL in [mapping/mapping.ttl](mapping/mapping.ttl).
- Department and branch remain SMIS resources even though the person resource is shared.
