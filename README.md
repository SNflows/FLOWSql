# Documentation: FLOWSql - FLOWS SQL query command-line client

`flowsql` is a command-line client for the SQL API of the [FLOWS](https://flows.phys.au.dk) supernova survey database. It lets you run SQL queries against the FLOWS database from your terminal, print the results in several formats, and run a handful of built-in helper queries (light curves, reference catalogs, table listings) without writing the SQL yourself.

This repo holds the binary files for FLOWSql.

---

## Setup

### Automated download and setup

```bash
sudo wget $(curl -sL https://install-scripts.bose.dev/detect-platform.sh | sh -s -- SNflows/FLOWSql flowsql) -O /usr/local/bin/flowsql && sudo chmod +x /usr/local/bin/flowsql
```
This will install the program appropriate for your CPU architecture, and make it system-wide available.

### Manual setup

One can simply download the binary and run it.

To download, go to the [release](https://github.com/SNflows/FLOWSql/releases) page and download the binary for your OS and architecture.

> Apple silicon users download `flowsql-darwin-arm64` binary.


If you want to make it available system-wide, run this in your terminal.


```bash
sudo mv path/to/flowsql-download-binary  /usr/local/bin/flowsql
sudo chmod 755 /usr/local/bin/flowsql
```


## API Authentication key

You can obtain the FLOWS API key from https://flows.phys.au.dk/myaccount.php, and **also** ask the FLOWS admins to enable you as a database user. Then set the `FLOWS_API_KEY=<key>` in your shell environment.

Every query needs your FLOWS API key, which `flowsql` reads from the `FLOWS_API_KEY` environment variable. There is no flag for it and no config file — if the variable is unset, the tool exits immediately with `Error: FLOWS_API_KEY not set`.

**Linux / macOS**

```bash
export FLOWS_API_KEY=<your key>
```

To make it permanent, add that line to `~/.bashrc` (bash) or `~/.zshrc` (zsh, the default on macOS), then open a new terminal.

**Windows**

In CMD, run:

```
rundll32.exe sysdm.cpl,EditEnvironmentVariables
```

and add `FLOWS_API_KEY` with your key as the value under the **User variables** section. You will need to reopen your terminal afterwards.

---

## Test your setup

This will print command line help
```bash
flowswl -h
```

This will connect to the database and run a simple query confirming `FLOWS_API_KEY` environment variable has been setup correctly.
```bash
flowsql -q "SELECT * FROM sites"
```

---

## Upgrading the binary

The FLOWSql binary is capable of self-updating if there is a newer version available, run

```bash
flowsql --upgrade
```

---

## Running a query

The core of the tool is `--query` (or `-q`):

```bash
flowsql --query "SELECT * FROM sites"
flowsql -q "SELECT * FROM targets LIMIT 10"
```

The query string is passed to the API as-is, so any SQL the server accepts will work — including joins, aggregates, and PostgreSQL-specific functions. The FLOWS database is PostgreSQL with the [Q3C](https://github.com/segasai/q3c) extension available, so spatial functions such as `q3c_radial_query` and `q3c_dist` can be used in your own queries too.

`flowsql` distinguishes two kinds of results automatically:

- **`select`** queries print rows in whichever output format you asked for.
- **write** queries (`INSERT`, `UPDATE`, `DELETE`) print a single line: `Query OK — N rows affected`.

If the API rejects the query, the error text from the server is printed and the tool exits with status 1.

### Admin mode

```bash
flowsql --admin -q "DELETE FROM refcat2 WHERE starid = 500001234500000"
```

`--admin` asks the server to run the query with elevated privileges. **This only works if your API key already has admin rights** on the SQL API — the flag by itself grants nothing. Admin mode is required for write operations, accessing some restricted tables, and is also useful for long-running `SELECT`s, since it bypasses the server-side query timeout.

Admin mode is **not** permitted together with `--askme`; if you combine them, `flowsql` warns you and silently drops back to non-admin.

---

## Output formats

Pick at most one. The default is the pretty table.

| Flag | Output |
|---|---|
| `--pretty` | Column-aligned text table with a row count (default) |
| `--json` | JSON array of row objects, indented |
| `--csv FILE` | Writes rows to `FILE` as CSV and prints the filename |
| `--raw` | One Go map per row, unformatted — mostly for debugging |

```bash
flowsql -q "SELECT * FROM sites LIMIT 5" --json
flowsql -q "SELECT * FROM sites LIMIT 5" --csv table.csv
```

The CSV writer uses the column order the API reports. If the API returns no column list, columns fall back to alphabetical order of the keys found in the rows. Empty and `NULL` values both come out as empty strings in table and CSV output; in JSON they stay `null`.

---

## Helper commands

These build a known-good query for you and run it through the same pipeline, so every output flag above applies to them as well.

### `--listtables`

Lists the table names in the `flows` schema.

```bash
flowsql --listtables
```

### `--listtable-cols TABLE`

Lists the column names of a table.

```bash
flowsql --listtable-cols targets
```

### `--getlc TARGET`

Fetches the light curve for a target by name. This is the query most users want and the most tedious one to write by hand: it joins `photometry_summary` against `files`, `targets`, `photometry_details`, and `sites`, drops photometry rows flagged as excluded, and returns observation time (as JD), filter, and the raw / subtracted / colour-term-corrected magnitudes with their errors, along with the site name and version.

```bash
flowsql --getlc 2021abc                         ## Display table on screen
flowsql --getlc 2021abc --csv 2021abc_lc.csv    ## Save the table to CSV file
```

Results are ordered by observation time and then filter.

### Reference catalog commands

The FLOWS database stores reference stars in `refcat2`. These commands operate on all stars within a **24 arcminute radius** (same as website) of the named target's coordinates.

**List catalog stars** — includes each star's angular distance from the target, sorted nearest first:

```bash
flowsql --catalog-list 2021abc
flowsql --catalog-list 2021abc --csv cat.csv
```

This query is heavy enough that it can hit the server's timeout; if it does, add `--admin` (assuming your key has admin rights) to bypass the limit.

**Delete catalog stars** for a target — requires `--admin`:

```bash
flowsql --admin --catalog-delete 2021abc
```

**Filter to custom stars only** — `--catalog-filter-custom` applies to both list and delete, and restricts the operation to stars in the custom-star ID range (`starid` between 5×10¹⁷ and 6×10¹⁷), i.e. those added via `--catalog-add` rather than coming from the original reference catalog. This is the safe way to undo your own additions without touching survey data:

```bash
flowsql --catalog-list 2021abc --catalog-filter-custom
flowsql --admin --catalog-delete 2021abc --catalog-filter-custom
```

⚠️ **`--catalog-delete` without `--catalog-filter-custom` deletes every reference star within 24′ of the target, including refcat2 catalog stars.** Run the equivalent `--catalog-list` first to see exactly what you are about to remove.

### `--catalog-add FILE` — adding custom catalog stars

Requires `--admin`. Reads a CSV file and inserts its rows into `refcat2` as custom stars.

```bash
flowsql --admin --catalog-add mystars.csv
```

The file's header line names the columns present; it may optionally begin with `#`. `ra` and `decl` are mandatory, every other column is optional, and any missing or empty value is inserted as `NULL`. All values must be numeric. 

As a shortcut, you can use `flowsql --catalog-list 2021abc --csv cat.csv` to dump a CSV and edit it, keeping only the required columns, and feed it back to `--catalog-add` input.

Allowed column names:

```
ra           (mandatory)
decl         (mandatory)
pm_ra
pm_dec
gaia_mag
gaia_bp_mag
gaia_rp_mag
J_mag
H_mag
K_mag
g_mag
r_mag
i_mag
z_mag
gaia_variability
V_mag
B_mag
u_mag
```

Example file:

```csv
#ra,decl,g_mag,r_mag,i_mag
183.2841,-12.4429,17.31,16.88,16.62
183.2903,-12.4512,18.02,17.55,
```

A few details worth knowing:

- **`starid` is generated for you.** Any `starid` or `distance` column in the file is ignored — this means the output of `--catalog-list` can be fed back into `--catalog-add` directly. Generated IDs start with `500...` followed by 15 more digits containing timestamp and a 5-digit serial number.
- Blank lines are skipped; a maximum of 100,000 rows can be added in one invocation.
- Column names are case-sensitive (so `J_mag`, not `j_mag`).
- If there is any mismatched column name, the query would fail and `flowsql` would report the error with a list of possible names — that error almost always means a typo in your column header.

The whole file becomes a single query, so it either all lands or none of it does. Validation happens locally first: a non-numeric value or a missing `ra`/`decl` fails with the offending line number before anything is sent.

---

## Asking questions in plain English

`--askme` sends a plain-text question to the FLOWSql AI service, which turns it into SQL by knowing the table schema, columns, and their physical meaning and interconnections. The generated SQL is printed and then executed:

```bash
flowsql --askme "list all ecsv files for all objects in flows project. also list filter. limit to 10 rows."
```

```bash
flowsql --askme 'list all files with filter H, with exposure time less than 500 second, which is a subtracted image, and is observed from the "NOT" telescope. return only first ten lines. along with file names list filter, exposure time, name of the site, target name, and redshift.'
```

Things to keep in mind:

- The generated SQL is echoed above the results so you can check it. With `--json` the echo is suppressed, keeping the output valid JSON.
- **`--admin` is force disabled for AI queries** — you will get a notice, and the query runs unprivileged. This is a security measure and deliberate design choice.
- So any write operations will print the generated query, but will fail to execute in `--askme` mode. You need to run the query manually with the `--admin` flag on.
- The request can take several seconds to a minute (GPU limit).
- If you see an authorization error from the AI server, run `flowsql --upgrade`; the AI credentials may become outdated for the old build.

Treat the generated SQL as a draft. It is good at exploratory questions, but read the echoed query before trusting a result you intend to publish.

---

## Using the API from Python

`--pycode` prints a self-contained Python class that talks to the same API, with no dependency on this CLI:

```bash
flowsql --pycode
```

Copy the printed block into your script:

```python
fsql = FLOWSql("FLOWS_API_KEY")
result = fsql.query("SELECT * FROM projects")
print(result['rows'])
```

`query()` returns a dict — `{"type": "select", "columns": [...], "rows": [...], "count": n}` for selects, `{"type": "write", "affected_rows": n}` for writes — and raises `FLOWSqlError` on any API or HTTP failure. It accepts the same `admin=True` argument as the CLI's `--admin` flag.

---

## Other options

| Flag | Purpose |
|---|---|
| `--examples` | Prints a short list of example invocations |
| `--upgrade` | Downloads and installs the latest release from [SNflows/FLOWSql](https://github.com/SNflows/FLOWSql) |
| `--url URL` | Overrides the API endpoint (default `https://flows.phys.au.dk/api/sqlquery.php`) |
| `-h`, `--help` | Full flag reference |

`--url` is only needed if you are pointed at a staging or mirror deployment; the default is correct for normal use.

---

## Quick reference

```bash
# Explore the schema
flowsql --listtables
flowsql --listtable-cols photometry_summary

# Light curve to CSV
flowsql --getlc 2021abc --csv 2021abc_lc.csv

# Free-form query
flowsql -q "SELECT target_name, redshift FROM targets ORDER BY redshift DESC LIMIT 20"

# Reference catalog round-trip
flowsql --catalog-list 2021abc --csv cat.csv
flowsql --admin --catalog-add newstars.csv
flowsql --admin --catalog-delete 2021abc --catalog-filter-custom

# Ask in English
flowsql --askme "how many targets are in the flows project?"
```
