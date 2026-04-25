# Contributing to Grotto Skills

Thanks for improving the Grotto skill ecosystem.

## Skill format

Each skill should live in:

```text
skills/<skill-name>/SKILL.md
```

Optional support files:

```text
skills/<skill-name>/templates/
skills/<skill-name>/references/
skills/<skill-name>/assets/
```

## PR checklist

- [ ] The skill has clear trigger conditions.
- [ ] The skill includes concrete steps or examples.
- [ ] The skill avoids private credentials and redacts tokens as `[REDACTED]`.
- [ ] The skill explains security boundaries if it touches identity, wallets, saves, or backend APIs.
- [ ] Templates or examples are small enough for creators to use directly.
