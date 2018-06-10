
## https://bugzilla.mozilla.org/show_bug.cgi?id=1466603

### comment I was going to post

Okay, now I think there's just a problem with VirtualBox (on 18.04).  I was experiencing a problem in output-file that looked like it could be similar short read problem (`thread 'main' panicked at 'index out of bounds: the len is 140 but the index is 140', /checkout/src/liballoc/vec.rs:1551:10`) so I updated the VM to 18.04 so it could be using the exact same version of Virtualbox, back to a short read variation where JS source fails to parse because of truncation:
```
/vagrant/scripts/js-analyze.js:358:30 SyntaxError: missing ) after argument list:
/vagrant/scripts/js-analyze.js:358:30         this.expression(stmt.r
/vagrant/scripts/js-analyze.js:358:30 ..............................^
```

Memtest86 seems to think my RAM is fine after 30 minutes and making it to test 9 (modulo 20), so I'm just going to try rsync-ing /vagrant over to disk inside the VM and running from there.

### general scenario
I've witnessed 3 types of failures:
1. Explicit short reads: run() failure in JS shell
2. Implicit short reads: JS syntax error on the root script that is clearly a
   truncation of the file
3. rust output-file bounds mismatch.

## other notes:

/vagrant/scripts/js-analyze.js:1221:1 Error: can't read /vagrant/sax/sax.js: short read
Stack:
  @/vagrant/scripts/js-analyze.js:1221:1
parallel: This job failed:




vagrant@ubuntu-1604-large-vmware:~/mozilla-config/mozilla-central$ jobs
[1]   Stopped                 vim setup
[3]   Stopped                 vim /vagrant/infrastructure/indexer-run.sh
[4]   Stopped                 vim /vagrant/infrastructure/indexer-setup.sh
[5]   Stopped                 vim -R build
[6]   Stopped                 vim /vagrant/scripts/mkindex.sh
[7]-  Stopped                 vim /vagrant/scripts/output.sh
[8]+  Stopped                 vim /vagrant/scripts/js-analyze.js
vagrant@ubuntu-1604-large-vmware:~/mozilla-config/mozilla-central$ ls /vagrant/scripts



/vagrant/infrastructure/indexer-run.sh ~/mozilla-config ~/mozilla-index mozilla-central


/vagrant/scripts/js-analyze.js:824:7 SyntaxError: missing ) after condition:
/vagrant/scripts/js-analyze.js:824:7   if (a
/vagrant/scripts/js-analyze.js:824:7 .......^



==> default: Checking for guest additions in VM...
    default: The guest additions on this VM do not match the installed version of
    default: VirtualBox! In most cases this is fine, but in rare cases it can
    default: prevent things such as shared folders from working properly. If you see
    default: shared folder errors, please make sure the guest additions within the
    default: virtual machine match the version of VirtualBox you have installed on
    default: your host and reload your VM.
    default:
    default: Guest Additions Version: 5.1.14
    default: VirtualBox Version: 5.2




==> default: Checking for guest additions in VM...
    default: The guest additions on this VM do not match the installed version of
    default: VirtualBox! In most cases this is fine, but in rare cases it can
    default: prevent things such as shared folders from working properly. If you see
    default: shared folder errors, please make sure the guest additions within the
    default: virtual machine match the version of VirtualBox you have installed on
    default: your host and reload your VM.
    default:
    default: Guest Additions Version: 5.1.34
    default: VirtualBox Version: 5.2
