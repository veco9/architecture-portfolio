# Design Doc: Dynamic SQL Query Builder

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Core Components](#core-components)
- [Security Considerations](#security-considerations)
- [Performance Optimizations](#performance-optimizations)
- [Testing Strategy](#testing-strategy)
- [Metrics](#metrics)
- [Related Documents](#related-documents)

---

## Overview

A database-agnostic query generation system for server-side data grids, supporting:
- Dynamic filtering, sorting, grouping, and pivoting
- Permission-aware query variants
- SQL injection protection via bind variables
- Oracle and PostgreSQL dialects

## Problem Statement

Enterprise data grids require complex, dynamic queries that:
1. Change based on user interactions (filters, sorts, groups)
2. Must respect user permissions (row-level security)
3. Need to work with legacy Oracle databases and modern PostgreSQL
4. Must prevent SQL injection while remaining flexible

ORM solutions often fall short for:
- Complex aggregations with CTEs
- Dynamic pivoting
- Database-specific optimizations
- Permission-based query modifications

## Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                      Query Builder Pipeline                   │
│                                                               │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────────────┐  │
│  │   Request   │   │   Query     │   │   Database          │  │
│  │   Parser    │──▶│   Builder   │──▶│   Executor          │  │
│  └─────────────┘   └─────────────┘   └─────────────────────┘  │
│        │                 │                     │              │
│        ▼                 ▼                     ▼              │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────────────┐  │
│  │ Filter      │   │ SQL         │   │ Result              │  │
│  │ Model       │   │ Template    │   │ Transformer         │  │
│  │ + Sort      │   │ + Binds     │   │                     │  │
│  │ + Group     │   │             │   │                     │  │
│  └─────────────┘   └─────────────┘   └─────────────────────┘  │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. Query Context

```python
@dataclass
class QueryContext:
    """Immutable context for query building."""

    # Grid request parameters
    filters: dict[str, FilterModel]
    sort_model: list[SortItem]
    group_cols: list[str]
    group_keys: list[str]
    pivot_cols: list[str]
    value_cols: list[AggregationCol]

    # Pagination
    start_row: int
    end_row: int

    # Permission context
    user_permissions: list[str]
    tenant_id: str

    # Database dialect
    dialect: Literal['oracle', 'postgresql']
```

### 2. Query Builder Interface

```python
from abc import ABC, abstractmethod

class QueryBuilder(ABC):
    """Base query builder interface."""

    @abstractmethod
    def build(self, context: QueryContext) -> QueryResult:
        """Build SQL query with bind variables."""
        pass

    @abstractmethod
    def get_base_cte(self) -> str:
        """Return base CTE for the query."""
        pass

    @abstractmethod
    def apply_permission_filters(self, context: QueryContext) -> str:
        """Apply row-level security based on permissions."""
        pass


@dataclass
class QueryResult:
    """Query result with SQL and bind variables."""
    sql: str
    bind_vars: dict[str, Any]
    count_sql: str  # Separate count query for performance
```

### 3. Filter Builder

```python
class FilterBuilder:
    """
    Translates AG Grid filter models to SQL WHERE clauses
    with bind variable protection.
    """

    def __init__(self, dialect: str, field_mapping: dict[str, str]):
        self.dialect = dialect
        self.field_mapping = field_mapping
        self._bind_counter = 0
        self._binds = {}

    def build(self, filter_model: dict) -> tuple[str, dict]:
        """
        Build WHERE clause from filter model.

        Returns:
            Tuple of (where_clause, bind_variables)
        """
        conditions = []
        for col_id, filter_config in filter_model.items():
            db_field = self.field_mapping.get(col_id, col_id)
            condition = self._build_condition(db_field, filter_config)
            if condition:
                conditions.append(condition)

        where_clause = ' AND '.join(conditions) if conditions else '1=1'
        return where_clause, self._binds

    def _build_condition(self, field: str, config: dict) -> str:
        filter_type = config.get('filterType')

        if filter_type == 'text':
            return self._build_text_condition(field, config)
        elif filter_type == 'number':
            return self._build_number_condition(field, config)
        elif filter_type == 'date':
            return self._build_date_condition(field, config)
        elif filter_type == 'set':
            return self._build_set_condition(field, config)
        return None

    def _build_text_condition(self, field: str, config: dict) -> str:
        operator = config.get('type')
        value = config.get('filter')
        bind_name = self._next_bind()

        mapping = {
            'equals': f"{field} = :{bind_name}",
            'notEqual': f"{field} != :{bind_name}",
            'contains': f"LOWER({field}) LIKE LOWER(:{bind_name})",
            'notContains': f"LOWER({field}) NOT LIKE LOWER(:{bind_name})",
            'startsWith': f"LOWER({field}) LIKE LOWER(:{bind_name})",
            'endsWith': f"LOWER({field}) LIKE LOWER(:{bind_name})",
            'blank': f"({field} IS NULL OR {field} = '')",
            'notBlank': f"({field} IS NOT NULL AND {field} != '')",
        }

        if operator in ('blank', 'notBlank'):
            return mapping[operator]

        # Prepare value based on operator
        if operator == 'contains' or operator == 'notContains':
            self._binds[bind_name] = f'%{value}%'
        elif operator == 'startsWith':
            self._binds[bind_name] = f'{value}%'
        elif operator == 'endsWith':
            self._binds[bind_name] = f'%{value}'
        else:
            self._binds[bind_name] = value

        return mapping.get(operator, f"{field} = :{bind_name}")

    def _build_set_condition(self, field: str, config: dict) -> str:
        """Build IN clause for set filters."""
        values = config.get('values', [])
        if not values:
            return '1=0'  # Empty set matches nothing

        bind_names = []
        for val in values:
            bind_name = self._next_bind()
            self._binds[bind_name] = val
            bind_names.append(f':{bind_name}')

        return f"{field} IN ({', '.join(bind_names)})"

    def _next_bind(self) -> str:
        self._bind_counter += 1
        return f'p{self._bind_counter}'
```

### 4. Permission-Aware Query Generation

```python
class PermissionQueryBuilder:
    """
    Generates different SQL based on user permissions.

    Example permission structures:
    - "region:north" → filter to northern region
    - "region:*" → no region filter (admin)
    - "org:sales/team1" → hierarchical access
    """

    def apply_permissions(
        self,
        base_sql: str,
        permissions: list[str],
        field_mapping: dict
    ) -> tuple[str, dict]:
        """
        Inject permission filters into query.

        Strategy:
        1. Parse permissions into categories
        2. For each category, add appropriate WHERE clause
        3. Wildcards (*) skip filtering for that category
        """
        binds = {}
        conditions = []

        # Group permissions by type
        perm_groups = self._group_permissions(permissions)

        for perm_type, values in perm_groups.items():
            if '*' in values:
                continue  # Admin access, no filter needed

            field = field_mapping.get(perm_type)
            if not field:
                continue

            # Build IN clause for permission values
            bind_names = []
            for i, val in enumerate(values):
                bind_name = f'perm_{perm_type}_{i}'
                binds[bind_name] = val
                bind_names.append(f':{bind_name}')

            conditions.append(f"{field} IN ({', '.join(bind_names)})")

        if conditions:
            # Wrap original query and add permission filter
            return f"""
                SELECT * FROM ({base_sql}) base_query
                WHERE {' AND '.join(conditions)}
            """, binds

        return base_sql, binds

    def _group_permissions(self, permissions: list[str]) -> dict[str, list[str]]:
        """Group permissions by type (e.g., region:north → {'region': ['north']})."""
        groups = {}
        for perm in permissions:
            if ':' in perm:
                perm_type, value = perm.split(':', 1)
                groups.setdefault(perm_type, []).append(value)
        return groups
```

### 5. CTE Manipulation

```python
class CTEBuilder:
    """
    Builds and manipulates Common Table Expressions.

    Used for:
    - Hierarchical queries
    - Multi-step aggregations
    - Permission injection points
    """

    def __init__(self):
        self.ctes: list[tuple[str, str]] = []  # (name, sql)

    def add_cte(self, name: str, sql: str) -> 'CTEBuilder':
        """Add a CTE to the chain."""
        self.ctes.append((name, sql))
        return self

    def build(self, final_select: str) -> str:
        """Build complete query with CTEs."""
        if not self.ctes:
            return final_select

        cte_parts = [f"{name} AS ({sql})" for name, sql in self.ctes]
        return f"WITH {', '.join(cte_parts)} {final_select}"


# Example usage for hierarchical data
class HierarchicalQueryBuilder:
    def build_tree_query(
        self,
        table: str,
        id_col: str,
        parent_col: str,
        root_ids: list[str]
    ) -> QueryResult:
        """Build recursive CTE for tree traversal."""

        if self.dialect == 'oracle':
            # Oracle uses CONNECT BY
            sql = f"""
                SELECT * FROM {table}
                START WITH {parent_col} IS NULL
                CONNECT BY PRIOR {id_col} = {parent_col}
            """
        else:
            # PostgreSQL uses recursive CTE
            sql = f"""
                WITH RECURSIVE tree AS (
                    SELECT *, 1 as depth
                    FROM {table}
                    WHERE {parent_col} IS NULL

                    UNION ALL

                    SELECT t.*, tree.depth + 1
                    FROM {table} t
                    JOIN tree ON t.{parent_col} = tree.{id_col}
                )
                SELECT * FROM tree
            """

        return QueryResult(sql=sql, bind_vars={}, count_sql='')
```

### 6. Dynamic Pivoting

```python
class PivotBuilder:
    """
    Generates dynamic pivot queries.

    Challenges:
    - Column names not known until runtime
    - Different syntax for Oracle vs PostgreSQL
    - Need to handle NULL values
    """

    def build_pivot(
        self,
        context: QueryContext,
        base_query: str,
        row_fields: list[str],
        pivot_field: str,
        value_field: str,
        agg_func: str = 'SUM'
    ) -> QueryResult:
        """
        Build dynamic pivot query.

        Oracle Example:
        SELECT * FROM (base_query)
        PIVOT (SUM(value) FOR pivot_col IN ('A' AS A, 'B' AS B))

        PostgreSQL Example:
        SELECT row_fields,
               SUM(CASE WHEN pivot_col = 'A' THEN value END) AS "A",
               SUM(CASE WHEN pivot_col = 'B' THEN value END) AS "B"
        FROM (base_query)
        GROUP BY row_fields
        """

        # First, get distinct pivot values
        pivot_values = self._get_pivot_values(base_query, pivot_field)

        if self.dialect == 'oracle':
            return self._oracle_pivot(
                base_query, row_fields, pivot_field, value_field, agg_func, pivot_values
            )
        else:
            return self._postgres_pivot(
                base_query, row_fields, pivot_field, value_field, agg_func, pivot_values
            )

    def _postgres_pivot(
        self,
        base_query: str,
        row_fields: list[str],
        pivot_field: str,
        value_field: str,
        agg_func: str,
        pivot_values: list[str]
    ) -> QueryResult:
        """PostgreSQL crosstab-style pivot."""
        case_columns = []
        binds = {}

        for i, val in enumerate(pivot_values):
            bind_name = f'pv_{i}'
            binds[bind_name] = val
            safe_alias = self._safe_column_name(val)
            case_columns.append(
                f'{agg_func}(CASE WHEN {pivot_field} = :{bind_name} '
                f'THEN {value_field} END) AS "{safe_alias}"'
            )

        sql = f"""
            SELECT {', '.join(row_fields)},
                   {', '.join(case_columns)}
            FROM ({base_query}) pivot_base
            GROUP BY {', '.join(row_fields)}
        """

        return QueryResult(sql=sql, bind_vars=binds, count_sql='')

    def _safe_column_name(self, value: str) -> str:
        """Sanitize value for use as column alias."""
        # Remove/replace problematic characters
        safe = ''.join(c if c.isalnum() or c == '_' else '_' for c in str(value))
        return safe[:30]  # Oracle max identifier length
```

### 7. Dialect Abstraction

```python
class SQLDialect(ABC):
    """Abstract base for database-specific SQL generation."""

    @abstractmethod
    def limit_offset(self, limit: int, offset: int) -> str:
        pass

    @abstractmethod
    def string_concat(self, *parts: str) -> str:
        pass

    @abstractmethod
    def coalesce(self, *expressions: str) -> str:
        pass

    @abstractmethod
    def date_trunc(self, unit: str, field: str) -> str:
        pass


class OracleDialect(SQLDialect):
    def limit_offset(self, limit: int, offset: int) -> str:
        return f"OFFSET {offset} ROWS FETCH NEXT {limit} ROWS ONLY"

    def string_concat(self, *parts: str) -> str:
        return ' || '.join(parts)

    def date_trunc(self, unit: str, field: str) -> str:
        return f"TRUNC({field}, '{unit}')"


class PostgreSQLDialect(SQLDialect):
    def limit_offset(self, limit: int, offset: int) -> str:
        return f"LIMIT {limit} OFFSET {offset}"

    def string_concat(self, *parts: str) -> str:
        return f"CONCAT({', '.join(parts)})"

    def date_trunc(self, unit: str, field: str) -> str:
        return f"DATE_TRUNC('{unit}', {field})"
```

## Security Considerations

### SQL Injection Prevention

```python
class QuerySanitizer:
    """
    Ensures all dynamic values use bind variables.
    Only allow-listed identifiers can be interpolated.
    """

    ALLOWED_IDENTIFIERS = re.compile(r'^[a-zA-Z_][a-zA-Z0-9_]*$')
    MAX_IDENTIFIER_LENGTH = 128

    def validate_identifier(self, identifier: str) -> bool:
        """Validate column/table name is safe for interpolation."""
        if not identifier:
            return False
        if len(identifier) > self.MAX_IDENTIFIER_LENGTH:
            return False
        return bool(self.ALLOWED_IDENTIFIERS.match(identifier))

    def sanitize_field_mapping(self, mapping: dict[str, str]) -> dict[str, str]:
        """Validate all field mappings are safe identifiers."""
        sanitized = {}
        for key, value in mapping.items():
            if not self.validate_identifier(value):
                raise ValueError(f"Invalid field identifier: {value}")
            sanitized[key] = value
        return sanitized
```

### Audit Logging

```python
class QueryAuditor:
    """Log all generated queries for security review."""

    def log_query(
        self,
        context: QueryContext,
        result: QueryResult,
        execution_time_ms: float
    ):
        log_entry = {
            'timestamp': datetime.utcnow().isoformat(),
            'tenant_id': context.tenant_id,
            'user_permissions': context.user_permissions,
            'query_hash': hashlib.sha256(result.sql.encode()).hexdigest()[:16],
            'filter_count': len(context.filters),
            'execution_time_ms': execution_time_ms,
            # Never log actual SQL or bind values in production
        }
        logger.info('query_executed', extra=log_entry)
```

## Performance Optimizations

```
┌─────────────────────────────────────────────────────────────────┐
│                 Query Optimization Checklist                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Index Hints                                                    │
│  • Generate index hints for Oracle when beneficial              │
│  • Use EXPLAIN ANALYZE in development                           │
│                                                                 │
│  Query Plan Caching                                             │
│  • Stable SQL structure → better plan cache hits                │
│  • Use bind variables, not string interpolation                 │
│                                                                 │
│  Count Optimization                                             │
│  • Separate count query without ORDER BY                        │
│  • Consider approximate counts for large tables                 │
│                                                                 │
│  Pagination                                                     │
│  • Keyset pagination for stable sort orders                     │
│  • Avoid deep OFFSET on large result sets                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Testing Strategy

```python
class QueryBuilderTests:
    """
    Test categories for query builder:
    1. Unit tests: Filter building, CTE manipulation
    2. Integration tests: Full query generation
    3. Security tests: Injection attempts
    4. Dialect tests: Oracle vs PostgreSQL output
    """

    def test_sql_injection_prevented(self):
        """Verify malicious input is parameterized."""
        filter_model = {
            'name': {
                'filterType': 'text',
                'type': 'equals',
                'filter': "'; DROP TABLE users; --"
            }
        }
        builder = FilterBuilder('postgresql', {'name': 'name'})
        where_clause, binds = builder.build(filter_model)

        # SQL should use bind variable, not interpolated value
        assert ':p1' in where_clause
        assert "DROP TABLE" not in where_clause
        assert binds['p1'] == "'; DROP TABLE users; --"

    def test_permission_filtering(self):
        """Verify permissions are enforced in query."""
        permissions = ['region:north', 'region:south']
        builder = PermissionQueryBuilder()

        sql, binds = builder.apply_permissions(
            "SELECT * FROM sales",
            permissions,
            {'region': 'region_code'}
        )

        assert 'region_code IN' in sql
        assert 'north' in binds.values()
        assert 'south' in binds.values()
```

## Metrics

- **Lines of code**: ~800 lines for core query builder
- **Supported dialects**: Oracle, PostgreSQL
- **Query generation time**: <10ms typical
- **Test coverage**: 90%+ on security-critical paths

## Related Documents

- [Server-Side Grid Patterns](../adr/fullstack/001-server-side-grid.md)
- [Analytics Caching](../adr/backend/003-analytics-caching.md)
