# Implementation Notes: Linux Re-export

Working notes, not document text.  Linux is one implementation
of re-export; nothing here should shape the problem statement or
the protocol design.  Read at v7.3-rc4 (tip 91db76f2e582 of
~/src/linux/server-development).

## Documentation

Documentation/filesystems/nfs/reexport.rst, "Reboot recovery":
when the re-export server reboots, the source server "is not in
grace, it cannot offer any guarantees that the file won't have
been changed between the locks getting lost and any attempt to
recover them.  The same applies to delegations and any associated
locks.  Clients are not allowed to get file locks or delegations
from a reexport server, any attempts will fail with operation not
supported."

## One flag gates locks and delegations

The NFS client's export operations set EXPORT_OP_NOLOCKS
(fs/nfs/export.c:165).  exportfs_cannot_lock() tests it
(include/linux/exportfs.h:311).  Callers:

- nfsd4_lock(), nfsd4_locku() (fs/nfsd/nfs4state.c:9497, :9844):
  NFS4ERR_NOTSUPP.
- nfs4_set_delegation() (fs/nfsd/nfs4state.c:7080): -EOPNOTSUPP;
  the OPEN succeeds with no delegation.
- lockd, via nlmsvc_file_cannot_lock(): four sites in
  fs/lockd/svclock.c (return values not read), and NLM_SHARE /
  NLM_UNSHARE in fs/lockd/svcshare.c:64, :112
  (nlm_lck_denied_nolocks).

## Share deny modes

- Not restricted on a re-export.  None of the deny-mode code
  tests the flag.
- Not propagated.  NFSD keeps deny modes in fi_share_deny and
  checks them in nfs4_file_check_deny() against opens through
  the same NFSD only.  The NFS client always sends share_deny = 0
  (encode_share_access(), fs/nfs/nfs4xdr.c:1413).

## Not confirmed

- Directory delegations: nfsd_get_dir_deleg() does not test the
  flag.  fs/nfs/dir.c has no setlease method, so the request
  would presumably fail in kernel_setlease(); fallthrough not
  read.
