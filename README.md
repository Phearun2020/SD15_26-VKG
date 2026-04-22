# HR-SMIS Knowledge Graph Demo

This project demonstrates integration between HR data and SMIS database data through a shared canonical link record.

## Flow

1. `HRPersonMap` reads the HR table in Dremio and creates one HR person resource:

```ttl
http://smis.itc.edu.kh/data/hr-person/{salary_id}
```

2. `CanonicalMap` joins HR and SMIS in Dremio and creates one canonical link resource:

```ttl
http://smis.itc.edu.kh/data/canonical/{hr_id}
```

3. `EmployeeMap` reads the SMIS side from Dremio and creates one employee resource:

```ttl
http://smis.itc.edu.kh/data/employee/{employee_id}
```

4. The graph is linked like this:

```text
hr_person -> canonical_record -> employee -> department / branch
```

5. The canonical resource stores the cross-system IDs:

```ttl
ex:canonicalId
ex:hrId
ex:smisId
```

## Dremio Credentials

Do not put Dremio credentials directly in the notebook or mapping file. Set them as environment variables before starting Jupyter.

```bash
export DREMIO_USER="your_username"
export DREMIO_PASSWORD="your_password"
export DREMIO_PASSWORD_URLENCODED="$(python -c 'from urllib.parse import quote_plus; import os; print(quote_plus(os.environ["DREMIO_PASSWORD"]))')"
export DREMIO_HOST="localhost"
export DREMIO_FLIGHT_PORT="32010"
```

`DREMIO_PASSWORD` is the raw password used by PyArrow Flight authentication.

`DREMIO_PASSWORD_URLENCODED` is used inside the SQLAlchemy `db_url`. It is needed because characters such as `@` must be encoded in URLs.

Example:

```text
raw password:       abc@123
URL-encoded value:  abc%40123
```

## Direct Morph-KGC Dremio Connection

The direct database connection is configured in `config.ini`:

```ini
[DataSource1]
mappings: mapping/mapping.ttl
db_url: dremio+flight://{DREMIO_USER}:{DREMIO_PASSWORD_URLENCODED}@{DREMIO_HOST}:{DREMIO_FLIGHT_PORT}/dremio?UseEncryption=false
```

The Dremio SQL queries are in `mapping/mapping.ttl`:

```ttl
rr:logicalTable [
  rr:sqlQuery """
    SELECT ...
    FROM "@phearun".hr h
    LEFT JOIN smis.public.users s
      ON LOWER(h.email) = LOWER(s.email)
    LEFT JOIN smis.public.employees e
      ON s.id = e.user_id
  """
] ;
```

The HR source is also defined in `mapping/mapping.ttl`:

```ttl
rr:logicalTable [
  rr:sqlQuery """
    SELECT *
    FROM "@phearun".hr
  """
] ;
```

## Run From Notebook

The notebook loads `../.env` automatically. You can either create `.env` from `.env.example` or start Jupyter from the same terminal where the environment variables are set:

```bash
jupyter notebook
```

Then run the Morph-KGC cell in `notebooks/hr-smis.ipynb`. It builds the Dremio `db_url` from environment variables and writes:

```text
output/output.ttl
```

## Run From Command Line

From the project root:

```bash
source .venv/bin/activate
python -m morph_kgc config.ini
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
- If the password contains special URL characters, update `DREMIO_PASSWORD_URLENCODED`.
- `canonical_link_fixed.json` is no longer the primary source.
- Update the HR SQL query in `mapping/mapping.ttl` if the Dremio HR table name changes.
- The canonical resource URI now uses `hr_id` as the stable key, while the generated cross-system identifier is stored as `ex:canonicalId`.
