Examples := $(wildcard *.json)
Manifest := manifest.cddl

all: schema examples

.PHONY: schema
schema: $(Manifest)
	@cddl $< g 1

.PHONY: examples
examples: $(Manifest) $(Examples)
	@for f in $(Examples); do \
		echo ">> validating $$f against $<" ; \
		cddl $< validate $$f || exit 1 ; \
	done
