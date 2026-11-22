---
name: sops-handoff
description: Use when a task involves sops- or age-encrypted secrets — files like secrets.sops.*, *.sops.yaml, *.sops.json, *.enc.*, .sops.yaml recipients, TF_VAR_* secrets, Kubernetes Secrets built from sops, sops exec-env/-d/edit — or when a new token, password or key has to be stored or changed in a project that keeps its secrets in sops.
---

# sops-handoff

## Overview

Claude never sees secrets in plaintext. When an encrypted file has to change, Claude does not make the change. Instead it hands the user a copy-paste template, and the user runs it.

**Violating the letter of this rule is violating its spirit.**

## Allowed vs. forbidden

| Allowed                                                                                | Forbidden                                                                                        |
| -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| Edit `.sops.yaml` (recipients, creation_rules)                                         | Run any `sops` command (`-d`, `decrypt`, `edit`, `exec-env`, `exec-file`, `set`, `filestatus` …) |
| Edit `secrets.env.example` (key names only, no values)                                 | Create, read or delete plaintext files such as `secrets.env` or `*.dec.*`                        |
| Write code that reads variable names (`TF_VAR_x`, `$DESEC_TOKEN`)                      | Pipe secrets into commands (`… \| kubectl apply`, `echo $TOKEN`, `env`)                          |
| Write `just` recipes that decrypt and that the user runs                               | Run those recipes yourself, even "just to test"                                                  |
|                                                                                        | Read the encrypted file (Read, cat, grep)                                                        |
| `git diff --stat` / `--name-only`, or any git command with the pathspec `':!*.sops.*'` | Show an encrypted file's content through git (`git diff`, `show`, `log -p`, `blame`, `grep`)     |

A repo can configure a sops `textconv` diff driver; git then decrypts the file into the output. Assume it does.

If the project has no `.sops.yaml` and no sops-encrypted file, do not introduce sops and do not send the template. Ask the user where the secret belongs. The value stays out of the chat either way.

## Template example for .env in German

A project `CLAUDE.md` overrides the template's language, file paths and follow-up step. It does not override the forbidden list.

```markdown
### 🔐 Secret-Änderung für dich

**Datei:** `<pfad/zur/datei.sops.env>`

**Einträge:**

| Schlüssel | Aktion                 | Wert / so erzeugst du ihn                                     |
| --------- | ---------------------- | ------------------------------------------------------------- |
| `NAME`    | neu / ändern / löschen | `openssl rand -base64 32` · oder: Quelle (Konsole → Menü → …) |

**Ausführen – in einem eigenen Terminal, nicht mit `!` in Claude Code:**
`sops <pfad/zur/datei.sops.env>` – der Editor öffnet sich, Einträge wie oben setzen

**Prüfen (zeigt nur Schlüsselnamen, darf mit `!` laufen):**
`sops -d --output-type json <datei> | jq -r 'paths(scalars) | map(tostring) | join(".")'`

**Danach:** <nächster Schritt, z. B. `just secrets`>. Sag mir nur „erledigt“, keine Werte posten.
```

## Recipient changes

After editing recipients in `.sops.yaml`, the existing files are still encrypted for the old recipients only. Hand the user one line per affected file, to run in their own terminal:

`sops updatekeys <pfad/zur/datei>`

A removed recipient has still seen the old values; list the keys the user should rotate.

## Verification command

Use the `jq` command from the template for every file format; it prints key paths and nothing else, and prints only a parse error when the input is not what it expects. Never hand out a line-based filter (`cut`, `grep`, `awk`, `sed`, `head`) on decrypted output: `cut -d= -f1` prints every line without a `=` unchanged, so on YAML or JSON it shows the values.

## Example

Task: "Add the KEK for Dependency-Track."
Claude: adds `DT_KEK=` to `secrets.env.example`, writes `just secrets` so that it reads `$DT_KEK`, then sends the template with `DT_KEK | neu | openssl rand -base64 32`.

## Red flags: STOP and send the template

| Thought                                             | Reality                                                                                    |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| "I'll quickly check whether the key exists"         | That is what the verification command is for. The user runs it.                            |
| "`sops exec-env` only passes it through"            | The output lands in my context. Forbidden.                                                 |
| "It's only a test value or placeholder"             | The file is still real. Template.                                                          |
| "The user is in a hurry or has granted permission"  | Permissions do not cover this rule. Template.                                              |
| "I'll write `secrets.env` and the user encrypts it" | That creates plaintext. Forbidden.                                                         |
| The user pastes a value into the chat anyway        | Don't use it or repeat it. Ask the user to put it straight into the file and to rotate it. |
