# Evidence for LineageOS issue #936

Issue: https://github.com/LineageOS/issues/issues/936

Device: Redmi / garnet (23122PCD1G)
ROM: 23.2-20261011-microG-garnet (build date 20261011, Android 16, security patch 2026-10-01)
Kernel: 5.10.269-gki-g5a95bbe47a3d
Bootloader: locked, verified boot green (ro.boot.verifiedbootstate=green, ro.boot.flash.locked=1)

## Summary

sys.user.0.ce_available is not set after a normal boot, so
StorageManagerService.mkdirs() rejects every request to create
/storage/emulated/0/Android/data/<pkg>/ and the framework throws

    java.lang.IllegalStateException: Failed to prepare /storage/emulated/0/Android/data/<pkg>/files

As a result no app can create its private external directories and
getExternalFilesDir() fails for app-specific dirs that do not exist yet.

## Files

* logcat-failure.txt - full stack of the exception (2026-10-25 19:02:50).
  The same block occurs 40 times in that capture; the server-side frame is
  com.android.server.StorageManagerService.mkdirs(StorageManagerService.java:3764).
* state-and-commands.txt - build props, sys.user.0.ce_available, workaround
  boot log, Android/data listing, ownership of the directories the system
  created after the property was set, dropbox boots/restarts, volume state.

## Evidence chain

1. At boot the property is empty. The workaround script logs the value it
   observes at boot (state-and-commands.txt, /data/adb/ce_fix.log):

       2026-10-25 19:16:39 start: ce_available=''
       2026-10-25 19:16:39 end:   ce_available='true'

2. While the property is empty, the exception in logcat-failure.txt is
   produced on every getExternalFilesDir() call for a missing directory.

3. After setting the property to true (setprop sys.user.0.ce_available true),
   the system itself creates Android/data/<pkg> for all installed apps, with
   the correct owner and mode (state-and-commands.txt):

       u0_a160 ext_data_rw 2770 ... /data/media/0/Android/data/com.google.android.youtube/
       u0_a88  ext_data_rw 2770 ... /data/media/0/Android/data/org.fdroid.fdroid/
       u0_a159 ext_data_rw 2770 ... /data/media/0/Android/data/org.telegram.messenger/
       u0_a165 ext_data_rw 2770 ... /data/media/0/Android/data/ru.aliexpress.buyer/
       u0_a158 ext_data_rw 2770 ... /data/media/0/Android/data/ru.oneme.app/

   No exception appears in logcat afterwards, and apps work normally.

4. Clearing the property again at runtime and waiting does not make anything
   re-set it; it stays empty until it is set manually or by the next boot.

5. Volumes are healthy (emulated;0 mounted), so this is not the FUSE/volume
   failure reported in #551. Verified boot is green and the bootloader is
   locked, so the ROM is installed normally.

6. dropbox shows the normal boots/restarts (state-and-commands.txt). The bug
   reproduces after a regular full reboot, not only after a system_server
   restart.

## Notes

* The 600 KB raw logcat of the failing capture (2026-10-25 19:02:50) and any
  additional captures (e.g. with the workaround disabled, root removed) can
  be provided on request.
* The workaround script is only mentioned for completeness; it is not needed
  to reproduce the problem. The failure occurs before any root script runs.
