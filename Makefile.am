# Makefile.am for the Python bindings of GPGME.
# Copyright (C) 2016 g10 Code GmbH
#
# This file is part of GPGME.
#
# GPGME is free software; you can redistribute it and/or modify it
# under the terms of the GNU Lesser General Public License as
# published by the Free Software Foundation; either version 2.1 of the
# License, or (at your option) any later version.
#
# GPGME is distributed in the hope that it will be useful, but WITHOUT
# ANY WARRANTY; without even the implied warranty of MERCHANTABILITY
# or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU Lesser General
# Public License for more details.
#
# You should have received a copy of the GNU Lesser General Public
# License along with this program; if not, see <https://gnu.org/licenses/>.
# SPDX-License-Identifier: LGPL-2.1-or-later

## Process this file with automake to produce Makefile.in

# Location of the released tarball archives.  This is prefixed by
# the variable RELEASE_ARCHIVE in ~/.gnupg-autogen.rc.  For example:
# RELEASE_ARCHIVE=user@host:archive/tarballs
RELEASE_ARCHIVE_SUFFIX = gpgme
# The variable RELEASE_SIGNKEY in ~/.gnupg-autogen.rc is used
# to specify the key for signing.  For example:
# RELEASE_SIGNKEY=D8692123C4065DEA5E0F3AB5249B39D24F25E3B6


# Autoconf flags
ACLOCAL_AMFLAGS = -I m4

EXTRA_DIST = \
	README \
	MANIFEST.in \
	VERSION \
	autogen.sh autogen.rc \
	gpgme.i \
	helpers.c helpers.h private.h \
	build-aux/libtool-patch.sed \
	doc/DCO doc/HACKING

if RUN_GPG_TESTS
tests = tests
else
tests =
endif

SUBDIRS = . ${tests} examples doc src

.PHONY: prepare
prepare: copystamp

# For VPATH builds we need to copy some files because Python's
# distutils are not VPATH-aware.
copystamp:
	ln -sf "$(srcdir)/src" gpg
	touch $@

all-local: copystamp
	set -e ; for PYTHON in $(PYTHONS); do \
	  CPP="$(CPP)" \
	  CFLAGS="$(CFLAGS)" \
	  srcdir="$(srcdir)" \
	  top_builddir="$(top_builddir)" \
	    $$PYTHON setup.py build --verbose --build-base="$$(basename "$${PYTHON}")-gpg" ; \
	done

python$(PYTHON_VERSION)-gpg/dist/gpg-$(VERSION).tar.gz.asc: copystamp
	$(MKDIR_P) python$(PYTHON_VERSION)-gpg-dist
	CPP="$(CPP)" \
	CFLAGS="$(CFLAGS)" \
	srcdir="$(srcdir)" \
	top_builddir="$(top_builddir)" \
	  $(PYTHON) setup.py sdist --verbose --dist-dir=python$(PYTHON_VERSION)-gpg-dist \
		--manifest=python$(PYTHON_VERSION)-gpg-dist/MANIFEST
	gpgbin=gpgconf --list-components | grep OpenPGP | sed -e 's/gpg:OpenPGP://g'
	$(gpgbin) --detach-sign --armor python$(PYTHON_VERSION)-gpg-dist/gpg-$(VERSION).tar.gz

.PHONY: sdist
sdist:	python$(PYTHON_VERSION)-gpg/dist/gpg-$(VERSION).tar.gz.asc

.PHONY: upload
upload: python$(PYTHON_VERSION)-gpg-dist/gpg-$(VERSION).tar.gz \
        python$(PYTHON_VERSION)-gpg-dist/gpg-$(VERSION).tar.gz.asc
	twine upload $^

CLEANFILES = copystamp \
	gpg

# Remove the rest.
#
# 'make distclean' clears the write bit, breaking rm -rf.  Fix the
# permissions.
clean-local:
	rm -rf -- build dist gpg.egg-info
	for PYTHON in $(PYTHONS); do \
	  find "$$(basename "$${PYTHON}")-gpg" -type d ! -perm -200 -exec chmod u+w {} ';' ; \
	  rm -rf -- "$$(basename "$${PYTHON}")-gpg" ; \
	done

install-exec-local:
	set -e ; for PYTHON in $(PYTHONS); do \
	  CPP="$(CPP)" \
	  CFLAGS="$(CFLAGS)" \
	  srcdir="$(srcdir)" \
	  top_builddir="$(top_builddir)" \
	  $$PYTHON setup.py \
	  build \
	  --build-base="$$(basename "$${PYTHON}")-gpg" \
	  install \
	  --prefix "$(DESTDIR)$(prefix)" \
	  --verbose ; \
	done

uninstall-local:
	set -x; \
	GV=$$(echo $(VERSION) | tr - _); \
	normalizedGV=$$(echo $$GV | sed s/_beta/b/); \
	for PYTHON in $(PYTHONS); do \
	  PLATLIB="$(prefix)/$$("$${PYTHON}" -c 'import sysconfig, os; print(os.path.relpath(sysconfig.get_path("platlib", scheme="posix_prefix"), sysconfig.get_path("data", scheme="posix_prefix")))')" ; \
	  rm -rf -- "$(DESTDIR)$${PLATLIB}/gpg" \
		"$(DESTDIR)$${PLATLIB}"/gpg-$$GV-py*.egg-info \
		"$(DESTDIR)$${PLATLIB}"/gpg-$$normalizedGV-py*.egg ; \
	done

dist-hook: gen-ChangeLog

distcheck-hook:
	set -e; ( \
	pref="#+macro: gpg-py_" ;\
	reldate="$$(date -u +%Y-%m-%d)" ;\
        echo "$${pref}ver  $(PACKAGE_VERSION)"  ;\
        echo "$${pref}date $${reldate}" ;\
        list='$(DIST_ARCHIVES)'; for i in $$list; do \
	  case "$$i" in *.tar.bz2) \
            echo "$${pref}size $$(wc -c <$$i|awk '{print int($$1/1024)}')k" ;\
	    echo "$${pref}sha1 $$(sha1sum <$$i|cut -d' ' -f1)" ;\
	    echo "$${pref}sha2 $$(sha256sum <$$i|cut -d' ' -f1)" ;;\
	  esac;\
	done ) | tee $(distdir).swdb

gen_start_date = 2015-05-05T00:00:00
gen-ChangeLog:
	if test -d $(top_srcdir)/.git; then				\
	  (cd $(top_srcdir) &&                      			\
	    $(GITLOG_TO_CHANGELOG) --append-dot --tear-off		\
	    --since=$(gen_start_date) ) > $(distdir)/cl-t;		\
          cat $(top_srcdir)/build-aux/git-log-footer >> $(distdir)/cl-t;\
	  rm -f $(distdir)/ChangeLog;					\
	  mv $(distdir)/cl-t $(distdir)/ChangeLog;			\
	fi


# Macro to help the release target.
RELEASE_NAME = $(PACKAGE_TARNAME)-$(PACKAGE_VERSION)

release:
	+(set -e;\
	 if [ "$(abs_top_builddir)" = "$(abs_top_srcdir)" ]; then \
           echo "error: build directory must not be the source directory" >&2;\
	   exit 2;\
         fi ;\
	 echo "/* Build started at $$(date -uIseconds) */" ;\
	 cd $(top_srcdir); \
	 ./autogen.sh --force; \
	 cd $(abs_top_builddir); \
	 rm -rf dist; mkdir dist ; cd dist ; \
	 $(abs_top_srcdir)/configure --enable-maintainer-mode; \
	 $(MAKE) distcheck; \
	 echo "/* Build finished at $$(date -uIseconds) */" ;\
         echo "/*" ;\
	 echo " * Please run the final step interactively:" ;\
	 echo " *   make sign-release" ;\
	 echo " */" ;\
	) 2>&1 | tee "$(RELEASE_NAME).buildlog"

sign-release:
	 +(set -e; \
	  test $$(pwd | sed 's,.*/,,') = dist || cd dist; \
	  x=$$(grep '^RELEASE_ARCHIVE=' $$HOME/.gnupg-autogen.rc|cut -d= -f2);\
          if [ -z "$$x" ]; then \
             echo "error: RELEASE_ARCHIVE missing in ~/.gnupg-autogen.rc">&2; \
             exit 2;\
          fi;\
          myarchive="$$x/$(RELEASE_ARCHIVE_SUFFIX)";\
	  x=$$(grep '^RELEASE_SIGNKEY=' $$HOME/.gnupg-autogen.rc|cut -d= -f2);\
          if [ -z "$$x" ]; then \
             echo "error: RELEASE_SIGNKEY missing in ~/.gnupg-autogen.rc">&2; \
             exit 2;\
          fi;\
	  mysignkey="$$x";\
	  files1="$(RELEASE_NAME).tar.bz2" ;\
	  files2="$(RELEASE_NAME).tar.bz2.sig \
		  $(RELEASE_NAME).swdb \
		  $(RELEASE_NAME).buildlog" ;\
	  echo "/* Signing the source tarball ..." ;\
	  gpg -sbu $$mysignkey $(RELEASE_NAME).tar.bz2 ;\
	  cat $(RELEASE_NAME).swdb >swdb.snippet;\
	  echo >>swdb.snippet ;\
	  sha1sum $${files1} >>swdb.snippet ;\
          cat "../$(RELEASE_NAME).buildlog" swdb.snippet \
               | gzip >$(RELEASE_NAME).buildlog ;\
          echo "Copying to local archive ..." ;\
	  scp -p $${files1} $${files2} $$myarchive/ || true;\
          echo "Uploading documentation ..." ;\
	  $(MAKE) -C doc online; \
	  echo '/*' ;\
	  echo ' * All done; for checksums see dist/swdb.snippet' ;\
	  echo ' */' ;\
         )
