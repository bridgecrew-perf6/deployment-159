# Backup mit Restic

## Restore mit Restic

Es gibt möglicherweise einen Mount /restore. Dieser ist beschreibbar.

### Auflisten Backup Inhalt

``` bash
restic ls {snapshotId}
```

### Wiederherstellen

Im Backup-Container:

```bash

/ # restic snapshots
repository f04178e7 opened successfully, password is correct
ID        Time                 Host                            Tags        Paths
----------------------------------------------------------------------------------
9db55305  2019-01-30 17:00:19  es-prod-backup-f9ff97c97-zl8jd              /backup
----------------------------------------------------------------------------------
1 snapshots

/ # restic restore 9db55305 --include /backup/mysqldump/  --target /restore/
repository f04178e7 opened successfully, password is correct
restoring <Snapshot 9db55305 of [/backup] at 2019-01-30 17:00:19.98690913 +0100 CET by @es-prod-backup-f9ff97c97-zl8jd> to ./
```

stellt das Verzeichnis "mysqldump" wieder her.

## Restore ElasticSearch

TODO: volle Wiederherstellung organisieren mit Scripts. Die Lösung unten funktioniert nur halb.
```bash
/restore/backup/elasticdump # elasticdump --input benutzer_booking_2019013011__data.json --output=https://geomap:Oq
3vITtm03@es-restore.stage2-all.geomap.immo/ --type=data
```
