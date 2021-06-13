####################################################################################################
# Configuration
####################################################################################################

# Build configuration

BUILD = build
MAKEFILE = Makefile
OUTPUT_FILENAME = bebook
CHAPTERS = chapters/*.md
TITLE=chapters/title.txt
TOC = --toc #--toc-depth=3
MATH_FORMULAS = --webtex
CSS_FILE = style.css
CSS_ARGS = --css=$(CSS_FILE)

# Combined arguments

ARGS = $(TOC) $(MATH_FORMULAS) $(CSS_ARGS) $(METADATA_ARGS) $(FILTER_ARGS) $(DEBUG_ARGS)
PANDOC_COMMAND = pandoc

# Per-format options

PDF_ARGS = -V documentclass=book --pdf-engine=xelatex -V geometry:"top=3cm, bottom=3cm, left=3cm, right=3cm" -V colorlinks -V urlcolor=Blue -V fontsize=8pt


####################################################################################################
# Basic actions
####################################################################################################

all:	book

book:	pdf

clean:
	rm -r $(BUILD)

####################################################################################################
# File builders
####################################################################################################

pdf:	$(BUILD)/pdf/$(OUTPUT_FILENAME).pdf

$(BUILD)/pdf/$(OUTPUT_FILENAME).pdf:	$(MAKEFILE) $(METADATA) $(CHAPTERS) $(CSS_FILE) $(IMAGES) $(TITLE)
	mkdir -p $(BUILD)/pdf
	$(PANDOC_COMMAND) $(ARGS) $(PDF_ARGS) -o $@ $(CHAPTERS) $(TITLE)
	@echo "$@ was built"
