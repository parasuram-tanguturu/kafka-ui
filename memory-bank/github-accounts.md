# GitHub Account Management

## Quick Reference

| Command | Description |
|---------|-------------|
| `gh auth status` | View all accounts |
| `gh auth switch -u <username>` | Switch active account |
| `gh auth login` | Add new account |
| `gh auth logout -h github.com -u <username>` | Remove account |

## Your Accounts

```
parasuram-tanguturu  ← default (kafka-ui fork)
swathi-uvce
sai-annamacharya
```

## Common Workflows

### Switch to a different account
```bash
gh auth switch -u swathi-uvce
```

### Verify current account before push
```bash
gh auth status | grep "Active account: true" -B2
```

### Push to fork (ensure correct account is active)
```bash
gh auth switch -u parasuram-tanguturu
git push -u myfork <branch-name>
```
