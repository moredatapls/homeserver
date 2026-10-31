# Paperless-ngx

## Temporary NumPy compatibility workaround

The HP N54L cannot run the x86-64-v2 baseline used by NumPy 2.4.
AI suggestions import NumPy through LlamaIndex and crash the web worker with
an illegal instruction when using the stock Paperless 3.4.0 dependencies.

The UI deployment installs the tested NumPy 2.3.5 wheel before executing the
image's normal `/init` entrypoint on every container start. It checks dependency
compatibility and runs a NumPy import and arithmetic check before starting
Paperless. A failed install or check stops startup instead of serving a broken
AI endpoint.

This keeps the official image but requires access to PyPI and its download hosts
at startup and adds installation time. The override lives in the container's
writable layer, not the document volumes. Keep the pin under review when
upgrading Paperless; remove the deployment's `command` and `args` override once
the image or host supports the required CPU baseline, or a compatible custom
image replaces this workaround.

## Restore procedure

1. Ensure that the directories on the host are empty
2. Get the backup link from Nextcloud and download it into `/mnt/data/paperless-ngx-secondary/sync/restore/restore.zip` (append `/download` to the share link to directly download the file)
3. Ensure that the Paperless-ngx version and the backup versions are the same, can be done as follows: `unzip -p backup.zip metadata.json | python3 -m json.tool`
4. Ensure that the file permissions are correct: `sudo chmod 0644 /mnt/data/paperless-ngx-secondary/sync/restore/restore.zip`
5. Run the document importer: `document_importer /usr/src/paperless/sync/restore/restore.zip --no-progress-bar`
6. You might have to create a new super user: `python manage.py createsuperuser`
