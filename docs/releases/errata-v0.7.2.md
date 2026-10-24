# Errata for v0.7.2-sc2

## Correction 1: the eMMC install path was verified

The v0.7.2-sc2 release note said the eMMC install path was not re-run for
this build. It was in fact installed to the eMMC during the release work,
and the box cold-booted without the stick and worked correctly. This
corrects the "not re-run" statements in the note's "What is still broken"
and "Last verified" sections.

The 24/24 smoke-test result recorded in the note was observed from the
stick boot. If an eMMC smoke-test result is wanted as well, run
`/root/smoke-test.sh` after the cold boot and add it here.
