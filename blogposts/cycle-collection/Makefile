
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

### graphviz (dot)
DOT := dot

DOT_SRCDIR := dot-src
DOT_OUTDIR := rendered

DOT_SRCFILES := $(wildcard $(DOT_SRCDIR)/*.dot)
# And their corresponding output file is rendered/FOO.svg
DOT_OUTFILES := $(patsubst %.dot,%.svg,$(patsubst $(DOT_SRCDIR)/%,$(DOT_OUTDIR)/%,$(DOT_SRCFILES)))

### Build Rules, extra deps
OUTFILES := $(MERMAID_OUTFILES) $(DOT_OUTFILES)
default: $(OUTFILES)

$(OUTFILES): Makefile

$(MERMAID_OUTFILES): seq.cfg
$(MERMAID_OUTFILES): $(MERMAID_OUTDIR)/%.svg: $(MERMAID_SRCDIR)/%
	$(MERMAID) --svg --css $(USE_CSS) --sequenceConfig seq.cfg --width $(WIDTH) --outputDir $(MERMAID_OUTDIR) $<

$(DOT_OUTFILES): $(DOT_OUTDIR)/%.svg: $(DOT_SRCDIR)/%.dot
	$(DOT) -Tsvg $< > $@

# The '-' prefix causes make to not care about the return value.  The -f arg to
# rm ensures it's not sad if the wildcard doesn't match anything.
clean:
	-rm -f $(MERMAID_OUTDIR)/* $(DOT_OUTDIR)/*

.PHONY: huh
huh:
	echo "Mermaid root is: $(MERMAID_ROOT)"
	echo "Mermaid using CSS: $(USE_CSS)"
	echo ""
	echo "DOT root is $(DOT_ROOT)"
