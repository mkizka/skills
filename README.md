# skills

自分用スキル集

## インストール

```bash
npx skills add mkizka/skills -y --skill '*' --agent claude-code -g
```

Claude Code のプラグインとして入れる場合

```bash
claude plugin marketplace add mkizka/skills
claude plugin install mkizka@mkizka-skills
```

## 動作確認

```bash
npx skills add . -y --skill '*' --agent claude-code -g
```

```bash
claude --plugin-dir .
```
