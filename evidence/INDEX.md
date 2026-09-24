# Evidence Index

This directory indexes evidence; it is not intended to become the bulk evidence store.

## Storage rule

Large or immutable artifacts normally remain in Dropbox, NAS storage, Codex investigation directories, device images, or other established evidence locations.

Git records only what materially improves retrieval and reasoning, such as:

- artifact name / evidence-set identity
- external canonical path
- SHA-256 or other verified identity when available
- date / scope
- short statement of what the artifact proves or contains
- relationship to a canonical Git document

## Provenance rule

A current Git conclusion should point back to the strongest underlying evidence when that provenance matters. Git documentation may supersede an older operational instruction without erasing the validity of the historical evidence that produced it.

## Do not

- copy huge captures or firmware into Git merely for convenience;
- duplicate the same evidence package across multiple storage systems without a reason;
- call normalized/reformatted material byte-verbatim evidence;
- treat absence from this index as proof that evidence does not exist during bootstrap.

This index will be populated incrementally as canonical documents are distilled and migrated.
