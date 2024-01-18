Examples := $(wildcard *.json)
Manifest := manifest.cddl

all: $(Manifest) $(Examples)
	@for f in $(Examples); do \
		echo ">> validating $$f against $<" ; \
		cddl $< validate $$f || exit 1 ; \
	done
