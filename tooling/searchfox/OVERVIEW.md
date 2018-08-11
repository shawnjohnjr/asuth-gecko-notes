## Web

### Routing
The nginx config covers this well.  Basically:

static:
- REPO/source

router.py:
- REPO/search
- REPO/define

rust:
- REPO/diff
- REPO/commit
- REPO/rev
- REPO/complete
- REPO/commit-info
