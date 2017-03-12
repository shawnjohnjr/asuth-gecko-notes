
### MERMAID
# You need to have mermaid.js on your path and you need to have node on your
# env.
MERMAID := mermaid.js

MERMAID_PIERCED := $(realpath $(shell which $(MERMAID)))
MERMAID_ROOT := $(abspath $(dir $(MERMAID_PIERCED))..)

MERMAID_SRCDIR := mermaid-src
MERMAID_OUTDIR := rendered

# Our sources files are src/FOO without extension.  If we include an extension,
# the CLI keeps it when it tacks ".svg" on the end.  (Which would be a friendly
# thing to fix and upstream.)
MERMAID_SRCFILES := $(wildcard $(MERMAID_SRCDIR)/*)
# And their corresponding output file is rendered/FOO.svg
MERMAID_OUTFILES := $(patsubst %,%.svg,$(patsubst $(MERMAID_SRCDIR)/%,$(MERMAID_OUTDIR)/%,$(MERMAID_SRCFILES)))

# mermaid.css or mermaid.FLAVOR.css. flavors: forest, dark
USE_CSS := $(MERMAID_ROOT)/dist/mermaid.css
WIDTH := 1400

### NOMNOML
# (And yes, I do know how to avoid the copy-paste stuff here, but that gets
# confusing fast.)
NOMNOML := nomnoml-cli.js
NOMNOML_PIERCED := $(realpath $(shell which $(NOMNOML)))
NOMNOML_ROOT := $(abspath $(dir $(NOMNOML_PIERCED))..)

NOMNOML_SRCDIR := nomnoml-src
NOMNOML_OUTDIR := rendered

NOMNOML_SRCFILES := $(wildcard $(NOMNOML_SRCDIR)/*)
# And their corresponding output file is rendered/FOO.svg
NOMNOML_OUTFILES := $(patsubst %,%.svg,$(patsubst $(NOMNOML_SRCDIR)/%,$(NOMNOML_OUTDIR)/%,$(NOMNOML_SRCFILES)))

### Build Rules, extra deps
OUTFILES := $(MERMAID_OUTFILES) $(NOMNOML_OUTFILES)
default: $(OUTFILES)

$(OUTFILES): Makefile

$(MERMAID_OUTFILES): seq.cfg
$(MERMAID_OUTFILES): $(MERMAID_OUTDIR)/%.svg: $(MERMAID_SRCDIR)/%
	$(MERMAID) --svg --css $(USE_CSS) --sequenceConfig seq.cfg --width $(WIDTH) --outputDir $(MERMAID_OUTDIR) $<

$(NOMNOML_OUTFILES): $(NOMNOML_OUTDIR)/%.svg: $(NOMNOML_SRCDIR)/%
	$(NOMNOML) $< $@

.PHONY: huh
huh:
	echo "Mermaid root is: $(MERMAID_ROOT)"
	echo "Mermaid using CSS: $(USE_CSS)"
	echo ""
	echo "nomnoml root is $(NOMNOML_ROOT)"
