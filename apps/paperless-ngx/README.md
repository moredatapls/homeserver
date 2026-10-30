# Paperless-ngx

## Restore procedure

1. Ensure that the directories on the host are empty
2. Get the backup link from Nextcloud and download it into `/mnt/data/paperless-ngx-secondary/sync/restore/restore.zip` (append `/download` to the share link to directly download the file)
3. Ensure that the Paperless-ngx version and the backup versions are the same, can be done as follows: `unzip -p backup.zip metadata.json | python3 -m json.tool`
4. Ensure that the file permissions are correct: `sudo chmod 0644 /mnt/data/paperless-ngx-secondary/sync/restore/restore.zip`
5. Run the document importer: `document_importer /usr/src/paperless/sync/restore/restore.zip --no-progress-bar`
6. You might have to create a new super user: `python manage.py createsuperuser`
