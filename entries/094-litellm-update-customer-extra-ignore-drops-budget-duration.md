# A request model omits a field its sibling endpoint accepts, and the base class's pydantic default discards it before any handler runs

- **Repo:** BerriAI/litellm
- **Surface:** `litellm/proxy/_types.py`, `UpdateCustomerRequest`, against its base
  `LiteLLMPydanticObjectBase` in `litellm/types/llms/base.py`, consumed by
  `litellm/proxy/management_endpoints/customer_endpoints.py::update_end_user` (`:541`)
- **Class:** message-conversion boundaries
- **Report:** reproduced publicly on
  [issue #33941](https://github.com/BerriAI/litellm/issues/33941#issuecomment-5017635020)
  (open), correcting the reporter's fault location. No PR from us: whether the field
  *should* be accepted on this endpoint is a product decision (`/customer/new` takes it,
  `/customer/update` does not document it), so the repair is the maintainers' to choose.

## Root cause

`POST /customer/update` binds its body to `UpdateCustomerRequest`, which declares
`user_id`, `alias`, `blocked`, `max_budget`, `budget_id`, `allowed_model_region`,
`default_model` and `object_permission`, and does not declare `budget_duration`. Its
base sets:

```python
class LiteLLMPydanticObjectBase(BaseModel):
    ...
    model_config = ConfigDict(protected_namespaces=())
```

No `extra="allow"`, so pydantic v2's default `extra="ignore"` applies and an undeclared
key is dropped at parse time, silently, with a 200 response. `update_end_user` then
builds its payload from `data.json()` at `customer_endpoints.py:541`, so
`budget_duration` is already gone before `non_default_values` and `budget_table_data`
exist.

The reporter placed the fault at `customer_endpoints.py:596-608`, the missing
`budget_reset_at` computation. That code is downstream of the drop, which changes what a
fix has to do: adding the `budget_reset_at` computation cannot help, because
`budget_duration` is never in `budget_table_data` to compute from. `budget_table_data` is
built once at `:575-582`, above the create/update branch, so raising the duration on an
already existing budget through this endpoint cannot work either. Accepting the field on
the request model is a precondition for any downstream repair.

Two details make the omission look deliberate rather than accidental, and both belong in
the report. `/customer/new` accepts `budget_duration` through `BudgetNewRequest`
(`_types.py:1649`) and documents it; the `/customer/update` docstring does not list it.
And the two classes defined immediately below the base in `types/llms/base.py`,
`BaseLiteLLMOpenAIResponseObject` and `HiddenParams`, both do set
`ConfigDict(extra="allow", ...)`. The permissive setting was available, was used twice
in the same file, and was not used here.

## Invariant violated

At a boundary that parses untrusted input into a typed model, the schema is the contract,
and a field absent from the schema is rejected or ignored, never forwarded. That much is
by design. The defect is what the boundary *reports*: `extra="ignore"` accepts the
request, returns success, and persists a subset of what the caller sent, so the client's
only evidence that anything was dropped is a later read of the stored record.

The rule worth carrying: silently ignoring unknown input is safe for a *reader* and
unsafe for a *writer*. On a write endpoint, a key the caller supplied and the server
discarded is data loss reported as success, and the two candidate policies
(`extra="forbid"`, which turns it into a 422, or declaring the field) are both louder
than the default. A write endpoint that inherits `extra="ignore"` from a shared base has
made that choice without stating it, and shared bases are exactly where such a choice
goes unexamined: it is set once, in a types module, and it governs every request model in
the service.

The second half of the invariant is about *families* of endpoints. When several endpoints
manage one resource, the union of fields any of them accepts is what a client will infer
is settable on all of them, because that is what the resource looks like from outside. A
field accepted by `create` and dropped by `update` is a real asymmetry whether or not it
was intended, so it needs to be either implemented or refused explicitly; leaving it to
the parser's default means the difference is invisible on both sides of the wire.

This is the same mechanism as [entry 041](041-giskard-pydantic-extra-ignore-drops-params.md)
in a different repo and a different direction of travel (there, parameters going out to a
provider; here, a management field coming in). Two independent instances suggest the
pydantic v2 default is a standing hazard wherever a model is used as a wire schema rather
than as an internal value type.

## Trigger

`POST /customer/update` with `budget_duration` in the body. The response is 200, the
customer's `max_budget` persists, and `budget_duration` and `budget_reset_at` stay null.

## Repro

Clean `python:3.12-slim`, dependencies from the released wheel with the HEAD tree ahead
of it on `sys.path`, imports confirmed to come from `/build/litellm/proxy/_types.py`, at
HEAD `5d4c4d0`, using the reporter's exact request body:

```
declared fields: ['alias', 'allowed_model_region', 'blocked', 'budget_id',
                  'default_model', 'max_budget', 'object_permission', 'user_id']
model_config extra = <unset -> pydantic default 'ignore'>
data.json() -> {'user_id': ..., 'alias': None, 'blocked': False, 'max_budget': 50.0,
                'budget_id': None, 'allowed_model_region': None, 'default_model': None,
                'object_permission': None}
non_default_values -> {'user_id': ..., 'max_budget': 50.0}

budget_duration survived request parsing: False
budget_duration reaches budget_table_data: False
```

**Scope of what was measured.** This is the request-parsing drop only. No proxy and no
database were stood up, so the persisted-row result remains the reporter's observation
and not ours; what these two lines establish is that the field is gone before any
database code is reached, which is the part that decides where the fix belongs.

Verified 2026-07-19 at HEAD `5d4c4d0`. Re-checked 2026-07-26: `UpdateCustomerRequest`
still omits `budget_duration`, and `LiteLLMPydanticObjectBase` still carries
`ConfigDict(protected_namespaces=())` with no `extra` setting, so the defect is live.
Not fixed upstream at the time of writing.
