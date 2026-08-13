# sagemath-skill

A [Claude Code skill](https://support.claude.com/en/articles/12512176-what-are-skills)
of recurring SageMath scripting pitfalls: stdout block-buffering when
redirected to a file or pipe (a killed job can silently lose everything
printed so far), the fix via `PYTHONUNBUFFERED` or explicit flush, and the
`sage -python`/`--python` trap that silently drops the Sage library and
preparser.

Every item here was hit and independently verified while running long
SageMath computations for a number-theory research project — this is a
distilled "don't step here again" checklist, not a general SageMath
tutorial. It's a companion to
[`pari-gp-skill`](https://github.com/iwaokimura/pari-gp-skill), split out
because the two tools' footguns don't overlap.

The skill itself lives in [`sagemath/SKILL.md`](sagemath/SKILL.md).

## Install

### Manual

```sh
git clone https://github.com/iwaokimura/sagemath-skill.git
cp -r sagemath-skill/sagemath ~/.claude/skills/
```

### As a Claude Code plugin

In Claude Code:

```
/plugin marketplace add iwaokimura/sagemath-skill
/plugin install sagemath@sagemath-skill
```

## License

MIT — see [LICENSE](LICENSE).
