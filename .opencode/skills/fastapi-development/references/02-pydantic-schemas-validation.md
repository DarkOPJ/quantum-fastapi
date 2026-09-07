# Pydantic Schemas & Validation

## Pydantic v2 baseline

- Use `field_validator` (not the deprecated v1 `@validator`) and `model_validator` for cross-field checks.
- Configure models via `model_config = ConfigDict(...)` (not the old `class Config:` inner class).
- Use `Annotated[Type, Field(...)]` for field-level constraints/metadata — the current idiomatic style over bare `Field()` defaults.

## Schema layering per resource

Define **separate schemas for separate purposes** rather than one model reused everywhere:

```python
class UserBase(BaseModel):
    email: EmailStr
    full_name: str

class UserCreate(UserBase):
    password: str  # write-only, never in a response schema

class UserRead(UserBase):
    id: int
    created_at: datetime
    model_config = ConfigDict(from_attributes=True)  # allows ORM -> schema mapping

class UserUpdate(BaseModel):
    email: EmailStr | None = None
    full_name: str | None = None  # all optional for PATCH semantics
```

- `UserCreate` never appears in a response; `UserRead` never accepts a password field. This separation is what prevents accidentally leaking sensitive fields (password hashes, internal flags) in an API response.
- `model_config = ConfigDict(from_attributes=True)` (formerly `orm_mode`) lets a response schema be constructed directly from a SQLAlchemy model instance.

## Validation patterns

- Prefer expressing constraints declaratively (`Annotated[str, Field(min_length=1, max_length=255)]`, `EmailStr`, `conint`/`Field(gt=0)`) over writing imperative `if` checks in a validator when possible — declarative constraints show up correctly in the generated OpenAPI schema, imperative checks don't.
- Use `field_validator` for anything declarative constraints can't express (cross-field consistency, normalization like lowercasing an email).
- Raise `ValueError` inside validators — FastAPI/Pydantic converts this into a well-formed 422 response automatically; don't catch and reformat it yourself in the common case.

## Settings & environment config

- `pydantic-settings`' `BaseSettings` is the standard way to load typed config from env vars/`.env`:

```python
class Settings(BaseSettings):
    database_url: str
    jwt_secret: str
    environment: Literal["dev", "staging", "prod"] = "dev"
    model_config = SettingsConfigDict(env_file=".env")

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

- Inject via `Depends(get_settings)` where needed rather than importing a module-level singleton everywhere — keeps it overridable in tests.

## `response_model` discipline

- Always set `response_model=` (or a return-type annotation FastAPI can use the same way) on every route — this is what enforces the "never leak an ORM model's extra fields" guarantee at the framework level, independent of what the service/repository actually returns.
- Use `response_model_exclude_none=True` sparingly and deliberately — it changes client-visible contract behavior; don't flip it on globally without checking downstream expectations.

## Definition of done for this phase
- [ ] Separate `Create`/`Update`/`Read` schemas per resource; no write-only fields (passwords, secrets) in any `Read` schema.
- [ ] Every route declares `response_model`.
- [ ] Config loaded via `pydantic-settings`, no raw `os.environ` reads scattered through the codebase.
- [ ] Validation uses declarative `Field`/`Annotated` constraints wherever possible, `field_validator` only for what declarative constraints can't express.
