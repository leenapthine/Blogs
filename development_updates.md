# Development updates for the DBSSIN library

**Author**: [Lee Napthine](/team/lee-napthine)  
**Last updated:** 2025-04-08

**Originally published at:** [https://arcsoft.uvic.ca/log/2025-02-28-developments-in-db-ingestion/](https://arcsoft.uvic.ca/log/2025-02-28-developments-in-db-ingestion/)

Over the last year, we’ve been adapting DBSSIN (Database-Spreadsheet-Ingestion) to better fit
current projects, and it’s come with its share of development challenges. The ARCsoft team has
been pushing the library forward, cleaning up the mapping system, and handling some unique
security and access issues.

If you haven’t been following along, we realized DBSSIN was going to be a necessary tool after
working on [ZooDB](https://gitlab.com/uvic-arcsoft/zoodb). This project relied
heavily on a spreadsheet ingestion process and we soon realized the widespread need for many of
our projects to ingest spreadsheet data into Django models without a bunch of manual cleanup.
Bhavy covered the early days in [this blog](https://arcsoft.uvic.ca/log/2024-05-25-generic-spreadsheet-ingestion/).
Now, it’s evolved into something bigger, as we engineer it for use in projects like the [Borders in Globalization](https://biglobalization.org/) Dyads Database (BIG), where
it helps securely load border relationship data into Django models.

## Legacy Issues:

Once we decided the proof of conceot was viable, there were a feww issues we knew we needed to
address:

1. First, we’d run into problems with inconsistent sheet names (extra spaces, typos,
   improper casing). The need to keep all input data uniform from client to client was not
   always possible and small errors in data were causing ingestion failures.

2. Second, DBSSIN originally mapped spreadsheet columns to Django fields using
   `help_text` in the model’s metadata. This hacky workaround hijacks a field
   meant for Django admin tooltips and suited our early proof of concept ingestion model,
   but was not suitable for long term development.

3. Finally, with sheets referencing cells in other sheets, we needed to figure out a way to
   create foreign keys that successfully mapped these relationships.

## The Sheet Mapping Problem:

We introduced a dedicated sheet attribute. This lets us explicitly define which sheet in the
spreadsheet maps to which Django model, instead of hoping names match up. The previous
functionality had led to some head scratching moments as the developer had to determine why
ingestion was failing.

Example:

```python
class Dyad(models.Model):
    """Model for the Dyads table"""

    sheet = "2.1.1.Dyads."

    dyad_ID = dbssin.IntegerField(primary_key=True, mapping="A", db_index=True)
    country_one = dbssin.CharField(max_length=64, mapping="B", null=False)
```

You may notice the new `dbssin.ValueFields` in our example. This is a new feature that
we implemented to address the following issue:

## Mixins: A Cleaner Way to Handle Mapping

We needed a way to map spreadsheet fields to Django models without overriding Django’s built-in
behavior. Hardcoding mappings in `help_text` wasn’t cutting it.

We introduced [Python mixins](https://www.pythontutorial.net/python-oop/python-mixin/)
to define mappings in a cleaner, reusable way. The hyperlink I shared does a pretty good job of
summarizing how to use mixins if they are unfamiliar. With mixins we created a unique set of
attributes for the DBSSIN library to map rows to fields in our models:

```python
class SheetMixin:
    """Mixin that adds a sheet attribute to a Django model"""

    def __init__(self, *args, sheet=None, **kwargs):
    self._sheet = sheet
    super().__init__(*args, **kwargs)

    @property
    def sheet(self):
    """Getter for sheet"""
    return self._sheet

    @sheet.setter
    def sheet(self, value):
    """Setter for sheet"""
    self._sheet = value


class MappingMixin:
    """Mixin that adds a mapping attribute to a Django model field"""

    def __init__(self, *args, mapping=None, **kwargs):
    self._mapping = mapping
    super().__init__(*args, **kwargs)

    @property
    def mapping(self):
    """Getter for mapping"""
    return self._mapping

    @mapping.setter
    def mapping(self, value):
    """Setter for mapping"""
    self._mapping = value
```

How it works in models:

```python
class CharField(MappingMixin, models.CharField):
    """CharField augmented with a mapping attribute"""
```

The integration into any project is simple. Import DBSSIN into your project and use the new
set of classes like you would any other field type. The end result:

1. We no longer override Django tool tips.
2. Mixins allow us to make the library more re-usable as it is carried over to new projects.
3. We achieve explicit control over how fields map to columns.

## Foreign Keys: When IDs Get Messy

For the BIG project, we had to reference related models via primary keys, but this caused issues
because:

- Some models had Django’s auto-generated IDs.
- Other models had natural primary keys (like `country_code`).
- Sometimes, IDs from different tables overlapped with real data values (e.g., an integer ID
  accidentally matching a population count).

In order to resolve these errors we require explicit primary keys for all referenced models. We
avoid relying on Django’s auto-generated IDs for foreign key references. As a result, we prevent
accidental mismatches and ensure foreign key relationships are properly mapped.

## Future Challenges: Row & Column Security and Handling Tiered Access

Currently, we are presented with an interesting problem when further developing DBSSIN for the
BIG project. The ‘big’ challenge: handling tiered access to data. Some spreadsheet
data needs to be fully public, while other parts are restricted to subscribers, internal
researchers, or admins. This means DBSSIN can’t just focus on ingestion. It has to
consider security at the database level too.

The issue boils down to two types of access control:

- Row-based access – Certain users should only see specific records.
- Column-based access – Some users should see only certain fields, even within records they
  can access.

For example, a department might see all employees in their unit but not their salaries, while
payroll staff might see salary data but not employees outside their department. Since DBSSIN is
responsible for ingestion, this filtering needs to happen at the database level, not in the
project frontend.

We’re considering two possible solutions:

- A permissions model that ties specific access levels to each field in a model.
- A dedicated permission sheet that gets ingested alongside the data, defining access rules
  dynamically.

Regardless of the approach, one thing is clear: security enforcement has to be server-side to
keep the database locked down. This is still evolving, but one way or another, DBSSIN is going
to handle access control as part of the ingestion process. Our analysis, while varied, agrees on
the need for a unique permissions class that operates on a inheritable
base model class with specific role-based CRUD (create, read, update, delete) methods that
cross-check the users permissions class.

### Integration with DBSSIN

How will this be integrated into DBSSIN? While nothing has been written in stone, some
suggestions are:

**A Role-Based Model:**

- Each model gets an associated permissions table.
- Fields are tagged with access levels.

**Dedicated Permission Sheets:**

- Users, groups, and permissions are defined in separate Excel sheets.
- These get ingested along with the data.

Each of these come with their own set of perks and challenges and prove that DBSSIN will be an
evolving library that changes shape as it is implemented into different projects. New problems
will arise and DBSSIN will continue to grow more robust as it adapts to resolve security and
ingestion specifications from each subsequent project.

## Final thoughts

DBSSIN keeps evolving, and these updates — better sheet mapping, mixin-based model mappings,
smarter foreign key handling, and row/column security — make it a flexible tool for data
ingestion.

We’re still working out the best way to handle role-based security, but one thing is clear:
server-side enforcement is non-negotiable.

Stay tuned for more updates as we refine security, improve error handling, and make DBSSIN even
more robust.

---
