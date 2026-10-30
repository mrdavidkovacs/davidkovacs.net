---
layout: post
title: "The Secret File Was Secure. The Application Could Not Read It."
date: 2026-10-30 07:00:00 +0100
categories: systems containers ansible security operations
excerpt: "A password file was rendered with strict permissions, but for the wrong owner. The fix preserved confidentiality and restored access only for the account that actually consumed it."
---

Paperless-ngx is a document archive with an ingestion pipeline. A new file first enters a consumption folder, can be adjusted by a pre-consume hook, and is then handed to the normal processing and OCR stages.

One source still delivered password-protected PDFs. The archive therefore used a small pre-consume script: it read a private list of known document passwords, removed the encryption from the working copy, and let Paperless continue normally.

Then one encrypted PDF stopped at the first step. The password file existed. Its mode was `0600`. It was deployed from encrypted configuration. On the surface, this looked like a correctly protected secret.

It was protected from the application as well.

## Problem

On October 28, 2026, the pre-consume script raised `PermissionError` while opening its password file:

```text
/usr/src/paperless/scripts/passwords.txt
```

The deployment had rendered that file as `root:root`, mode `0600`. The Paperless runtime account used UID and GID `1000`. With `0600`, only the file owner can read it. Root could read the file; the process that needed it could not.

Paperless runs a configured pre-consume script after it sees a document and before processing begins.[1] In this installation, that hook removed PDF password protection before Paperless continued with the document. A password file that the hook cannot read turns that stage into a hard failure.

The requirement was not simply “keep the file private”. It was more precise:

- keep the password list unreadable to unrelated accounts;
- allow the Paperless worker identity to read it;
- change only this one rendered secret file;
- avoid weakening permissions for the scripts directory or other secrets.

A secret file has both confidentiality and availability requirements. `0600` satisfies neither requirement on its own. The owner must be the identity that needs the file.

## Possible approaches

### Relax the mode

Changing the file to `0644`, or making its directory broadly readable, would make the immediate error disappear. It would also expose a password list to accounts that do not need it.

That is not a repair. It replaces one access-control mistake with a wider one.

### Run the hook as root

The file was already readable by root. Changing the application path so that one helper runs with more privilege would make the ownership mistake less visible, while making the runtime model more complicated.

The process already had a dedicated service identity. The secret should be granted to that identity instead.

### Make this one file owned by the worker

The worker account is the intended reader. Rendering this one password file as UID and GID `1000`, while retaining mode `0600`, gives exactly that account read and write access and nobody else any access.

This is also the smallest infrastructure change. The deployment continues to render the secret from the same encrypted input. Only the resulting file metadata changes.

## Solution

The stack configuration now overrides the ownership of this one rendered file:

```yaml
stack_secret_files:
  - path: scripts/passwords.txt
    owner: "1000"
    group: "1000"
    mode: "0600"
```

Ansible supports setting file owner, group and mode explicitly, including numeric owner and group identifiers.[2]

The important detail is that the mode did not become less strict. The change made `0600` apply to the right account.

The password file remains private to the Paperless runtime identity. Other accounts still cannot read it. The pre-consume script now runs with access to the one secret it needs.

## Decrypting PDFs before Paperless consumes them

Paperless documents this extension point directly: a pre-consume script receives `DOCUMENT_WORKING_PATH`, the copy of the document that Paperless will process. A script that changes a document should change only that working copy.[1]

For a Docker deployment, mount a scripts directory into the container and point `PAPERLESS_PRE_CONSUME_SCRIPT` at the executable script:

```yaml
services:
  webserver:
    volumes:
      - ./scripts:/scripts:ro
    environment:
      PAPERLESS_PRE_CONSUME_SCRIPT: /scripts/decrypt.py
```

The decryption script in this installation uses `pikepdf` and a password list with one candidate per line. The important parts are the input path from Paperless, `allow_overwriting_input=True`, and failing when none of the configured passwords works:

```python
#!/usr/bin/env python3
import os
from pathlib import Path

import pikepdf

document = os.environ["DOCUMENT_WORKING_PATH"]
if not document.lower().endswith(".pdf"):
    raise SystemExit(0)

try:
    with pikepdf.open(document) as pdf:
        if not pdf.is_encrypted:
            raise SystemExit(0)
except pikepdf.PasswordError:
    pass

passwords = Path("/scripts/passwords.txt").read_text().splitlines()
for password in passwords:
    try:
        with pikepdf.open(document, password=password,
                          allow_overwriting_input=True) as pdf:
            pdf.save(document)
        break
    except pikepdf.PasswordError:
        continue
else:
    raise RuntimeError("no configured password could open this PDF")
```

The password list belongs outside the repository and must be readable only by the container identity that runs the hook. This example assumes an image that already contains `pikepdf`; otherwise, build a small derived image with that dependency rather than installing packages into a running container.

`qpdf` is a viable alternative when there is one known password: its `--password-file` option reads the password from a file and `--decrypt` writes an unencrypted output file.[3] In either case, test with a disposable encrypted PDF first, then verify the consumed document in Paperless. Never place a password directly in Compose, a command line, or version control.

## Verification

The configuration change was deployed successfully on October 29, 2026.

Runtime verification checked the deployment from the application's side:

1. the rendered password file was `0600` and owned by `1000:1000`;
2. the Paperless worker account could read it;
3. the Paperless container was healthy;
4. an encrypted PDF that had previously failed was re-consumed successfully.

The last step matters. A successful configuration run proves that the file was rendered. A successful read as the worker proves the Unix permission boundary. Re-consuming the previously failed document proves that the real consumer path can use the password list rather than merely inspect it.

## Why this solution holds up

The fix states the actual rule: a secret belongs to the runtime identity that consumes it.

It does not make the whole scripts directory readable. It does not loosen other secret files. It does not add a privileged execution path. It does not depend on a manual `chown` surviving the next deployment.

The deployment configuration remains the source of truth for both the file contents and the access boundary. Every later rollout recreates the same narrow rule: Paperless can read its password list; other accounts cannot.

This distinction is easy to lose when reviewing secret files. A restrictive mode looks reassuring, but a secret that its application cannot read is not correctly deployed. It is only unavailable.

## Limitation

This configuration deliberately assumes that UID and GID `1000` remain the Paperless runtime identity. If the container image or deployment changes that identity, the file ownership must change with it and the worker-read check must be repeated.

The rule should not become a blanket default for rendered secrets. Each secret should be owned by the narrowest runtime identity that actually needs it.

## References

1. [Paperless-ngx: consumption hooks](https://docs.paperless-ngx.com/advanced_usage/#pre-consume-script)
2. [Ansible: `ansible.builtin.file` module](https://docs.ansible.com/projects/ansible/13/collections/ansible/builtin/file_module.html)
3. [qpdf: command-line options](https://qpdf.readthedocs.io/en/stable/cli.html)
