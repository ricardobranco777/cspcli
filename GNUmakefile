# SPDX-License-Identifier: BSD-2-Clause

ifeq ($(shell id -u),0)
PREFIX  ?= /usr/local
else
PREFIX  ?= $(HOME)
endif
bindir  ?= $(PREFIX)/bin

SCRIPTS = aws az gcloud

INSTALL         = install
INSTALL_PROGRAM = $(INSTALL) -m 755
LN_S            = ln -sf

.PHONY: all install uninstall test

all:

install:
	mkdir -p "$(DESTDIR)$(bindir)"
	$(INSTALL_PROGRAM) $(SCRIPTS) "$(DESTDIR)$(bindir)/"
	$(LN_S) gcloud "$(DESTDIR)$(bindir)/gsutil"

uninstall:
	cd "$(DESTDIR)$(bindir)" && rm -f $(SCRIPTS) gsutil

test:
	shellcheck -e SC1090,SC1091,SC2174 $(SCRIPTS)
